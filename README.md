# hmon
A simple client for the upcoming (v19.0) Dyalog APL Health Monitor.

To play with it, start a couple of v19.0 interpreters with HMON_INIT using different
port numbers:

HMON_INIT="SERVE:localhost:7001"

Then from APL:

            Init
            Connect 'localhost' 7001
            Connect 'localhost' 7002
            conns
      1  CLT00000000  localhost  7001
      2  CLT00000001  localhost  7002
      
            'receivepoll' 5000∘GetFacts¨( 1 'Workspace')(2 'Workspace')


