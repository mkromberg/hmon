# hmon
A simple client for the upcoming (v19.0) Dyalog APL Health Monitor.

To play with it, start a couple of v19.0 interpreters with HMON_INIT using different
port numbers:

Run the function Demo, and then start one or more APL systems using:

dyalog HMON_INIT="POLL:localhost:7000" LOAD="c:\devt\hmon\aplsource\DemoApplication.aplf"

