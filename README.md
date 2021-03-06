fulltrace
=========

A complete ftrace- and uprobes-based tracer (user, libraries, kernel) for GNU/Linux

fulltrace traces the execution of an ELF program, providing as output a full trace of its userspace, library and kernel function calls. This is achieved by analyzing the ELF structure of the user-provided executable to identify all the function included in its original code. Then, the focus of the inspection moves to the libraries used by the executable (the dynamic dependencies of the executable, those listed with `ldd`); fulltrace plants userspace debug probes (uprobe) on the address of these userspace symbols. The ftrace kernel subsystem is therefore notified when those symbols are encountered and is able to include them in a full function_graph trace.
In an attempt to restrict the tracing activity to the bare execution of the user-provided program, fulltrace encapsulates it in a wrapper, whose source is available along with the main full-trace.sh script.

Table of contents:

1. [Prerequisites](#prerequisites)
2. [Configuration](#configuration)
3. [Command-line interface](#command-line-interface)
4. [Usage examples](#usage-examples)
5. [Output file](#output-file)
6. [Author](#author)

Prerequisites
-------------

* fulltrace relies on a Linux kernel compiled with its built-in tracing subsystem (ftrace), the function\_graph tracer and transparent user-space probes. In more detail, the following kernel configuration options and their dependencies must be set as enabled (=y): FTRACE, TRACING\_SUPPORT, UPROBES, UPROBE\_EVENT, FUNCTION\_GRAPH\_TRACER.
* fulltrace also needs the `objdump`, `readelf` and `ldd` executables to be installed.
* As it is necessary to compile the wrapper program, the `gcc` compiler, the `ld` linker and the `make` utility must be available.
* Root privileges are required to run the full-trace.sh script.

Configuration
-------------

Before starting to use the full-trace.sh script, the wrapper program needs to be compiled. It is necessary that the user issues the following command inside the `fulltrace` directory:  
`$ make`

Command-line interface
----------------------

In its full-trace.sh main script, fulltrace provides a straight-forward command-line interface to its features. Details about its usage can be printed by running it with its `-h` option.

<pre>
$ ./full-trace.sh -h
usage: ./full-trace.sh
&#9;[-b|--bufsize bufsize\] [-c|--clean] [-d|--debug] [-h|--help]
&#9;[-o|--output] [-t|--trace] [-u|--uprobes]
&#9;[-k|--ksubsys subsys1,...] [-i|--i-config]
&#9;[-s|--save-uprobes file] [-l|--load-uprobes file]
&#9;-- <command> <arg>...

Full process tracer (userspace, libraries, kernel)
OPTIONS:
-b|--bufsize&#9;&#9;Set the per-cpu buffer size (KB)
-c|--clean&#9;&#9;Clean temporary files
-d|--debug&#9;&#9;Debug output
-h|--help&#9;&#9;This help
-o|--output&#9;&#9;Output decoding
-t|--trace&#9;&#9;Process tracing
-u|--uprobes&#9;&#9;Uprobes creation
-k|--ksubsys&#9;&#9;Enable traces for listed subsystems
-i|--i-config&#9;&#9;Ignore kernel configuration check
-s|--save-uprobes&#9;&#9;Save created uprobes also in the specified file
-l|--load-uprobes&#9;&#9;Use an alternate file for uprobes
</pre>

Usage examples
--------------

The user, to begin with, might want to obtain a trace from scratch; it is then necessary for the fulltrace script to perform the following steps:

1. analyze the executable and plant the userspace debug probes for all the symbols its execution involves;
2. trace the execution of the process;
3. decode the output trace to trim spurious uprobe events and substitute function names to the bare addresses of the symbol.

As explained by the usage help, this means passing the `-u` (uprobes creation), `-t` (process tracing) and `-o` (output decoding) options to the full-trace.sh script. Use the following command line to **obtain a full trace from scratch**.  
`$ ./full-trace.sh -uto -- <executable_name> [executable_arguments]`

The **output file** generated by fulltrace is stored in the `/tmp` folder; its name is `trace.decoded`. Note that every run of fulltrace overwrites the output file.

Use the following command line to **cleanup temporary files** stored by fulltrace in the `/tmp` folder, (only the trace.decoded file will be preserved).  
`$ ./full-trace.sh -c`

At the beginning of its execution, fulltrace will attempt to check the configuration of the running kernel. In case no valid configuration is found (that is, no `/proc/config.gz` or `/boot/config-<running_kernel_version>` file is found), fulltrace will exit with an error. In case a valid configuration is found for the currently running kernel, fulltrace will check if every needed option is enabled, and, if any necessary option is disabled, exit with a list of the missing symbols.  
It is however possible to skip this step by passing the `-i` option to the script. Use the following command line to **ignore the kernel configuration** and fully trace a command.  
`$ ./full-trace.sh -utoi -- <executable_name>`  
Please note that, if no tunable of the uprobes subsystem is found in the debugfs, it won't be possible to skip the kernel configuration check.

The most time-expensive operation performed by fulltrace is the analysis of user-provided executable and needed libraries. This operation may be performed only once per session, if the user wants to re-use the same set of userspace probes. However, if the same uprobes set is used to trace two different programs, the program-specific functions of the second executable will not be included in the trace.  
This kind of usage is convenient when:

* the user doesn't care about program-specific functions, just wants a call trace of library functions, and both the programs use the same libraries;
* the user wants to trace the the execution of the same program more than once.

Use the following command line to **generate and plant uprobes** for a program.  
`$ ./full-trace.sh -u -- <executable_name> [executable_arguments]`  
This next command line will trace the execution of a program **using an already generated set of userspace probes**.  
`$ ./full-trace.sh -to -- <executable_name> [executable_arguments]`

To avoid this time-expensive task, the user may also want to save the generated uprobes to a file and afterwards re-load them. This can be obtained by using the `-s` and `-l` options.  
Use the following command line to **save uprobes to a file**; in this example, the uprobes will be saved to the `saved.uprobes` file. Note that the `-s` option also forces the creation of a new set of uprobes. The `-s` option may also be used in conjunction with other options (this simple example won't specify other actions).  
`$ ./full-trace.sh -s saved.uprobes -- <executable_name> [executable_arguments]`  
This next command line will, on the contrary, **load uprobes from a file**. If the specified file does not exist, fulltrace will force the generation of a new uprobes set with a warning, therefore however allowing the user to complete other tasks, if specified; this example shows the usage of the `-l` option (loading uprobes from the `saved.uprobes` file) along with the `-t` (tracing) and `-o` (output decoding) options.  
`$ ./full-trace.sh -l saved.uprobes -to -- <executable_name> [executable_arguments]`

By default, the total amount of memory assigned to the ftrace subsystem as a tracing buffer is 1/4 of the currently available memory. To change this behavior, the user may use the `-b` option.  
Use the following command line to **specify the per-CPU tracing buffer** to be assigned to the ftrace subsystem. In this example, a buffer of 1MB will be assigned to each CPU. Note that the value is expressed in KB (1MB = 1024 KB).  
`$ ./full-trace.sh -uto -b 1024 -- <executable_name> [executable_arguments]`

Output file
-----------

The format of the output file is based on the classic function\_graph provided by ftrace, that shows a call graph of the traced functions. The ftrace kernel subsystem if configured by fulltrace to include, for each event:

* absolute timestamps (the funcgraph-abstime option);
* execution context of the event, that is the process/thread that originally issued the function call (the funcgraph-proc option);
* execution latency of kernel functions (as provided by the function\_graph tracer).

The basic ftrace output is improved by fulltrace to show, along with the execution latency of kernel functions, the duration of userspace and library functions.

Author
------

Please address any bugreport or comment to the author:  
>Mauro Andreolini (<mauro.andreolini@unimore.it>)
