---
layout: post
section-type: post
title: Modifying Android Xamarin assemblies bundled into a .so file
category: reverse-engineering
tags: [ 'mono', 'android', 'xamarin', 'reverse engineering' ]
---

## Introduction

Assemblies bundling is a security measurement available in the enterprise edition of Visual Studio.
Basically, when enabled, .NET assemblies get gzipped and bundled into a native shared library.
This makes it harder to edit an assembly or even perform static analysis and also reduces the size of the apk.
The post will mainly focus on how to bundle assemblies back into a native library.
{% lightbox /img/posts/mono-rebundle/bundling_example1.png --data="img1" --title="The native library contains .NET assemblies" --alt="Mono Bundle" %}

## Unpack APK

First off in order to get access to native libraries, an apk has to get unpacked.
This can be done using any Zip decompression tool though I use apktool https://ibotpeaches.github.io/Apktool/ myself.
After decompressing native libraries can be found inside /libs/[ABI]/ directory.

{% lightbox /img/posts/mono-rebundle/apk_unzipped_libs.png --data="img2" --title="Decompressed APK" --alt="Unzipped APK" %}

## Extract assemblies

The file we are interested in is libmonodroid_bundle_app.so, this is where bundled assemblies are kept.
There are currently available utilities to extract assemblies from a native library. 
I personally use this one https://github.com/tjg1/mono_unbundle and can recommend it to you without hesitation.
Installation is quick and requires Python >= 3.5 and Pipenv.<br/>
The assemblies will most likely lose their original name, so you will need to fix it manually.

{% lightbox /img/posts/mono-rebundle/extracted_dlls.png --data="img3" --title="Extracted assemblies before and after rename" --alt="Rename DLLs" %}

## Analyze

Having the assemblies extracted, any .NET assembly editor can be used from this point. I will be using dnSpy https://github.com/0xd4d/dnSpy.<br/>
Go to File -> Open, select an assembly and voila there is its source code that we can perform static analyze on.

{% lightbox /img/posts/mono-rebundle/dnSpy_opened_assembly.png --data="img4" --title="App1.dll opened in dnSpy" --alt="dnSpy" %}

## Edit

DnSpy also allows to edit an existing assembly.<br/>
Let's say we have a simple activity with a button and a text showing how many times the button was clicked.
This is actually how the activity from the previous section works. <br/>
We want to change the behavior so the clicks counter gets increased not by 1 but by 5 each click.

{% lightbox /img/posts/mono-rebundle/dnSpy_edit_count1.png --data="img5" --title="The button click event handler" --alt="Edit count" %}

There is an option called "Edit IL Instructions" in dnSpy that will allow you to edit the handler.

{% lightbox /img/posts/mono-rebundle/dnSpy_edit_count2.gif --data="img6" --title="Modifying the handler" --alt="dnSpy edit IL" %}

Simple as that, don't forget to save the assembly (File -> Save Module).

## (Re)bundle
This is the moment we need to bundle the assemblies back to a native library.
I couldn't find any utility to help me do it but I imagine a person with appropriate knowledge can achieve that with the mkbundle tool which is delivered with Mono.
Anyway, I lack such knowledge but having in mind that Xamarin is open source and it bundles assemblies automatically for us, I was able to find the corresponding code and based on that create a bundling utility.
The utility is available at GitHub repo https://github.com/met94/Mono-Rebundle. The repo contains information about how to use it, so go there for the details.

The bundling process starts with generating a stub. Run the utility with the following command line parameters:
<pre><code data-trim>
Mono_Rebundle.exe produce-stab --path C:\assemblies
</code></pre>
This will produce .\temp\bundles\<ABI>\temp.c which contains a template with exports for the native library. <br/>
Since the template differs from the original code found in libmonodroid_bundle_app.so, manual adjustments need to be made.<br/>
Probably I use mono_mkbundle that was not meant to be used for Android Xamarin and that's the reason why my template doesn't match. Anyway, manual patching works so I didn't bother to investigate further.

The original mono_mkbundle_init:
<pre><code data-trim class="c">
int __fastcall mono_mkbundle_init(int (__fastcall *a1)(int), int a2)
{
  char *(**v2)[2]; // r3
  size_t size; // ST18_4
  int v4; // ST14_4
  _DWORD *dest; // ST08_4
  int v7; // [sp+0h] [bp-34h]
  int (__fastcall *v8)(int); // [sp+4h] [bp-30h]
  int v9; // [sp+Ch] [bp-28h]
  void *v10; // [sp+10h] [bp-24h]
  int i; // [sp+1Ch] [bp-18h]
  _DWORD *v12; // [sp+20h] [bp-14h]
  char *(**v13)[2]; // [sp+24h] [bp-10h]
  const void **j; // [sp+24h] [bp-10h]

  v8 = a1;
  v7 = a2;
  install_dll_config_files();
  v13 = compressed;
  for ( i = 0; ; ++i )
  {
    v2 = v13;
    ++v13;
    if ( !*v2 )
      break;
  }
  bundled = (int)malloc(4 * (i + 1));
  v12 = (_DWORD *)bundled;
  for ( j = (const void **)compressed; *j; ++j )
  {
    size = *((_DWORD *)*j + 2);
    v4 = *((_DWORD *)*j + 3);
    v10 = malloc(*((_DWORD *)*j + 2));
    v9 = my_inflate(*((_DWORD *)*j + 1), v4, v10, size);
    if ( v9 )
    {
      fprintf((FILE *)((char *)&_sF + 168), "mkbundle: Error %d decompressing data for %s\n", v9, *(_DWORD *)*j, v7);
      exit(1);
    }
    *((_DWORD *)*j + 1) = v10;
    dest = malloc(0xCu);
    memcpy(dest, *j, 0xCu);
    *dest = *(_DWORD *)*j;
    *v12 = dest;
    ++v12;
  }
  *v12 = 0;
  return v8(bundled);
}
</code></pre>
As you can see the original one gets two parameters of which the first is a pointer to mono_api.mono_register_bundled_assemblies and the second is unused.<br/><br/>
Patched mono_mkbundle_init:
<pre><code data-trim class="c">
void __fastcall mono_mkbundle_init(int (__fastcall *a1)(int), int a2)
{
	CompressedAssembly **ptr;
	MonoBundledAssembly **bundled_ptr;
	Bytef *buffer;
	int nbundles;

	install_dll_config_files ();

	ptr = (CompressedAssembly **) compressed;
	nbundles = 0;
	while (*ptr++ != NULL)
		nbundles++;

	bundled = (MonoBundledAssembly **) malloc (sizeof (MonoBundledAssembly *) * (nbundles + 1));
	if (bundled == NULL) {
		// May fail...
		mkbundle_log_error ("mkbundle: out of memory");
		exit (1);
	}

	bundled_ptr = bundled;
	ptr = (CompressedAssembly **) compressed;
	while (*ptr != NULL) {
		uLong real_size;
		uLongf zsize;
		int result;
		MonoBundledAssembly *current;

		real_size = (*ptr)->assembly.size;
		zsize = (*ptr)->compressed_size;
		buffer = (Bytef *) malloc (real_size);
		result = my_inflate ((*ptr)->assembly.data, zsize, buffer, real_size);
		if (result != 0) {
			fprintf (stderr, "mkbundle: Error %d decompressing data for %s\n", result, (*ptr)->assembly.name);
			exit (1);
		}
		(*ptr)->assembly.data = buffer;
		current = (MonoBundledAssembly *) malloc (sizeof (MonoBundledAssembly));
		memcpy (current, *ptr, sizeof (MonoBundledAssembly));
		current->name = (*ptr)->assembly.name;
		*bundled_ptr = current;
		bundled_ptr++;
		ptr++;
	}
	*bundled_ptr = NULL;
	a1((const MonoBundledAssembly **) bundled);
}
</code></pre>
After patching temp.c to contain the differences, use the utility to link into libmonodroid_bundle_app.so.
<pre><code data-trim>
Mono_Rebundle.exe link
</code></pre>
This will produce a new libmonodroid_bundle_app.so containing the modified assemblies. Put it back in the APK, install it & profit.