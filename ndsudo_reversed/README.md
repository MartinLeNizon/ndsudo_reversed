> This report explained, from a revere engineering perspective, how `ndsudo` executable from `netdata` versions prior to `1.45.2` works. It was the binary used when exploiting `CVE-2025-24893`.

# Analysis

The binary first considers `--help` and `--test`. In the case of `--test`, a variable is left to `1` (instead of `0` otherwise). 

```c
first_arg = (char *)argv[1];
if ( !strcmp(first_arg, "--help") || !strcmp(first_arg, "-h") )
{                                         // help
	show_help();
	exit(0);
}
if ( !strcmp(first_arg, "--test") )
	first_arg = (char *)argv[2];            // test
else
	is_test = 0;
	command = find_command(first_arg)
```

`find_command()` retrieves the correct structure in memory based on user option. As an example, if the user does `./ndsudo nvme-list`, it will retrieve the following structure, where `user_command = "nvme-list"`, `executable = "nvme"` and `args = list --output-format=json`.

```
00000000 struct ndsudo_command // sizeof=0x20
00000000 {
00000000     char *user_command;
00000008     char *args;
00000010     char *executable;
00000018     _QWORD gap;
00000020 };
```

Then, it searches if the executable, from the hardcoded structure, is present with executable rights in one of the directories specified in the `PATH` environment variable. If found, it will execute the first binary found with correct arguments got from the structure above.

# Conclusion

`ndsudo` resolves user commands (such as `nvme-list`) into an executable name (`nvme`) followed by arguments (`list --output-format=json`), and will execute the first binary found in the `PATH` environment variable.


```
(C) Netdata Inc.

A helper to allow Netdata run privileged commands.

  --test
    print the generated command that will be run, without running it.

  --help
    print this message.

The following commands are supported:

- Command    : nvme-list
  Executables: nvme 
  Parameters : list --output-format=json

- Command    : nvme-smart-log
  Executables: nvme 
  Parameters : smart-log {{device}} --output-format=json

- Command    : megacli-disk-info
  Executables: megacli MegaCli 
  Parameters : -LDPDInfo -aAll -NoLog

- Command    : megacli-battery-info
  Executables: megacli MegaCli 
  Parameters : -AdpBbuCmd -aAll -NoLog

- Command    : arcconf-ld-info
  Executables: arcconf 
  Parameters : GETCONFIG 1 LD

- Command    : arcconf-pd-info
  Executables: arcconf 
  Parameters : GETCONFIG 1 PD

The program searches for executables in the system path.

Variables given as {{variable}} are expected on the command line as:
  --variable VALUE

VALUE can include space, A-Z, a-z, 0-9, _, -, /, and .
```