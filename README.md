# UXY adds structure to the standard UNIX tools

Treating everything as a string is the way through which the great power and
verstatility of UNIX tools is achieved. However, sometimes the constant
parsing of strings gets a bit cumbersome.

UXY is a tool to manipulate the UXY format, which is a basically
a two-dimenstional table that's both human- and machine-readable.

Additionally, UXY wraps some common UNIX tools and exports their output in
UXY format. Along with convertors from/to other common data formats
(e.g. JSON) this allows for quick and painless access to the data.

### Examples

```
$ uxy ls .
PERMISSIONS LINKS OWNER GROUP SIZE TIME NAME 
-rw-rw-r--  1     sustrik sustrik 4147 "2019-05-19 17:50:13.263709560 +0200" README.md 
-rwxrwxr-x  1     sustrik sustrik 8021 "2019-05-19 17:48:14.962014184 +0200" uxy
```

```
$ uxy ls . | uxy align
PERMISSIONS LINKS OWNER   GROUP   SIZE TIME                                  NAME
-rw-rw-r--  1     sustrik sustrik 4147 "2019-05-19 17:50:13.263709560 +0200" README.md 
-rwxrwxr-x  1     sustrik sustrik 8021 "2019-05-19 17:48:14.962014184 +0200" uxy
```

```
$ uxy ls . | uxy reformat "NAME SIZE"
NAME SIZE 
README.md 4147 
uxy  8021 
```

```
$ uxy ls . | uxy to-json
[
    {
        "LINKS": "1",
        "NAME": "README.md",
        "TIME": "2019-05-19 17:50:13.263709560 +0200",
        "OWNER": "sustrik",
        "GROUP": "sustrik",
        "SIZE": "4147",
        "PERMISSIONS": "-rw-rw-r--"
    },
    {
        "LINKS": "1",
        "NAME": "uxy",
        "TIME": "2019-05-19 17:48:14.962014184 +0200",
        "OWNER": "sustrik",
        "GROUP": "sustrik",
        "SIZE": "8021",
        "PERMISSIONS": "-rwxrwxr-x"
    }
]
```

```
$ uxy ps | uxy to-json | jq '.[].CMD'
"bash"
"uxy"
"uxy"
"jq"
"ps"
```

# UXY format

### Records and fields

- UXY format is a possibly infinite list of lines separated by newlines.
  Each line is a data record.
- Each line is composed of fields separated by arbitrary number spaces.
- Fields starting AND ending with a double quote are treated in a special way:
  - The delimiting quote characters themselves are ignored.
  - The characters inside the quotes, including spaces, are taken as they are.
  - The only exception are the characters preceded by backslash. These are
    encoded as follows:
    - \" double quote
    - \\ backslash
    - \t tab
    - \n newline
- UXY format should not contain control characters, such as TABs.
  If there's a need for a TAB, use "\t" instead.

### Header

- First line of UXY format is the header.
- The format of the header line is identical to any other UXY line,
  however, its semantics differ.
- The value of a field in the header is the name of that particular column.
- The spacing of the fields in the header is a hint for the tools. They SHOULD
  try to align the data with the headers.

### Interactions between headers and records

- Fields SHOULD but don't have to be aligned with each other or with the
  headers.
- If there's less fields in a record than in the header the missing values
  are assumed to be empty strings.
- It's valid to for a record to have more fields than the header.
  The tools should consider the name of such column to be an empty string.

Example:

```
NAME  AGE ADDRESS
Alice 25  "Main Road 1, London" "Let's use this unnamed field for comments."
Bob   23  ""
Carol 55  "Hotel \"Excelsior\", New York"
```

# TOOLS

All UXY tools take input from stdin and write the result to stdout.

### uxy re

Reads the lines of the input and parses each one using the supplied regular
expression. Matched groups are then assigned to the fields specified in the header.

```
$ ls -l | uxy re "TIME NAME" ".* +(.*) +(.*)"
TIME NAME 
14:28 README.md
14:22 uxy
```

### uxy align

Aligns the data with the headers. This is done by resizing the columns so that even
the longest value fits into the column.

```
$ ls -l | uxy re "TIME NAME" ".* +(.*) +(.*)" | uxy align
TIME  NAME
14:36 README.md 
14:22 uxy
```

This command doesn't work with infinite streams.

### uxy reformat

Takes an UXY input and reformats it according to the supplied headers.

It allows for:

- reordering of columns
- resizing of columns
- dropping columns
- adding new columns

```
$ cat test.uxy 
TIME  NAME
15:03 README.md 
16:08 uxy
$ uxy reformat "NAME          TIME" < test.uxy 
NAME          TIME 
README.md     15:03 
uxy           16:08 
```

### uxy from-json

Converts from JSON to UXY format.

```
$ cat test.json 
[
    {
        "Name": "Quasimodo",
        "Time": "14:30"
    },
    {
        "Name": "Moby Dick",
        "Time": "14:22"
    }
]
$ uxy from-json < test.json 
Name        Time
Quasimodo   14:30 
"Moby Dick" 14:22
```

### uxy to-json

Convers UXY format to JSON.

```
$ ls -l | uxy re "time name" ".* +(.*) +(.*)" | uxy to-json
[
    {
        "time": "14:22",
        "name": "README.md"
    },
    {
        "time": "14:22",
        "name": "uxy"
    }
]
```

### uxy ls

Wraps ls tool and outputs the results in UXY format.

```
$ uxy ls . | uxy align
PERMISSIONS LINKS OWNER   GROUP   SIZE TIME                                  NAME
-rw-rw-r--  1     sustrik sustrik 3535 "2019-05-19 16:21:56.835218781 +0200" README.md 
-rwxrwxr-x  1     sustrik sustrik 7455 "2019-05-19 17:33:49.286905670 +0200" uxy
```

### uxy ps

Wraps ps tool and outputs the results in UXY format.

```
$ uxy ps | uxy align
PID   TTY    TIME     CMD
12392 pts/22 00:00:00 bash 
28221 pts/22 00:00:00 uxy
28222 pts/22 00:00:00 uxy
28223 pts/22 00:00:00 ps
```

TODO: Wrap more common UNIX tools.

