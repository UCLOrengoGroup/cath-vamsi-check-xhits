# Checking for cross-hits in CATH sequence datasets

## Overview

We have a dataset of sequences from different clusters (from different superfamilies in CATH).
We want to use this dataset to train/test a neural network that will discriminate between sequences
from different superfamilies. 

The first step is to check that this dataset does not include any crosshits (ie significant sequence 
similarity from different superfamilies).

## Dataset

We start with a dataset of sequences that have been assigned to a 'class' (ie the structural cluster id, which includes the superfamily assignment).
These have already been clustered to remove any completely identical sequences.

```
$ head -n 5 S100_dataset.csv
,Unnamed: 0,Sequence,SSG5_Class,FunFam
0,0,PEDLVRAEDLSKYRQVASHAGLHSASVPGILSLDVCPADTNKVLTGGADKNVVVFDKSEEQIVATLKGHTKKVTSVIYHPSQSVVFSASPDTTIRVWSVTGGNCVQVVRAHEAGVTGLSLHATGDYLLSSSEDQYWAFSDIQTGKVLTKVTDESAGCALTCAQFHPDGLIFGTGTADSQIKIWDLKERTNVANFPGHSGPVTSIAFSENGYYLATGAQDSSLKLWDLRKLKNFKTITLDNNYEVKSLVFDQSGTYLAVAGSDIRVYICKQWSEVLNFTDHTGLVTGVAFGEHAKFLTSAGMDRSLRFYSL,2.130.10.10-SSG5-21,43
32,32,PEELVRAEDLSKYRQVASHAGLHSASVPGILALDLCPSDTNKVLTGGADKNVVVFDKNEEQIVATLKGHTKKVTSVIFHPSQSVVFSASPDTTIRVWSVTAGNCVQVVRAHEAGVTGLSLHATGDYLLSSSEDQYWAFSDIQTGRVLTKVTDESAGCALTCAQFHPDGLIFGTGTADSQIKIWDLKERTNVANFPGHSGPVTSIAFSENGYYLATGAQDSSLKLWDLRKLKNFKTITLDNNYEVKSLVFDQSGTYLAVGGSDIRVYICKQWSEVLNFTDHSGLVTGVAFGENAQFLTSAGMDRSLKFYSL,2.130.10.10-SSG5-21,43
64,64,TLAPIDAIESYTQISSHPFHKTNKQGIISLDILYSKDLIATGGIDTNAVIFDRPSGQILSTLSGHSKKVTSVKFVAQGESVLTGSSDKTVRLWQRSDDGNYNCRHILKDHTAEVQAVTVHATNNFFVTASLDGSWCFYELSSGTCLTQVFDTSGSSEGYTSAAFHPDGLILGTGTTESLVKIWDVKSQANVARFDGHAGPVTAISFSENGYFLATAAHDGVKLWDLRKLKNFRNFAPYDSETPTNSVEFDHSGSYLAIAGSDIRIYQVANVKSEWNCVKTFPDLSGTGKATCVKFGADSKYIAVGSMDRNLRIFGL,2.130.10.10-SSG5-21,43
```

## Steps to check for Xhits

Install BLAST

```
sudo apt install ncbi-blast+
```

Create FASTA sequence file

```
tail -n +2 S100_dataset.csv | awk -F, '{print ">"$4"__FF"$5"__"$1"\n"$3}' > S100_dataset.fasta
``` 

Create BLAST database

```
makeblastdb -in S100_dataset.fasta -dbtype prot
```

Run all versus all BLAST

```
blastp -db S100_dataset.fasta -query S100_dataset.fasta -outfmt 6 > S100_allvall.blast6.out
```

Standard BLAST columns

```
qaccver saccver pident length mismatch gapopen qstart qend sstart send evalue bitscore
```

Get top 100 xhits from hits between sequences from different superfamilies

Explanation of steps below:
- change SSG ids to superfamily ids
- only show results where sfam ids differ, the seqid > 40 and evalue < 1e-3
- sort by seqid
- only show the one hit per sfam1/sfam2 xhit

```
perl -pe 's/([\d.]+)-(\S+)/$1/mg' S100_allvall.blast6.out \
	| awk '$1 != $2 && $3 > 40 && $11 < 1e-3 {print}' \
	| sort -k3,3fr \
	| sort -k1,2 -u \
	> S100_sfam_xhits.uniq.blast6.out
```

The BLAST job is still running, but this currently results in 57 xhits between SSGs in different superfamilies, eg


```
$ cat S100_sfam_xhits.uniq.blast6.out
1.10.1160.10    3.40.50.620     44.898  49      26      1       5       52      264     312     7.47e-05       42.0
1.10.1170.10    3.30.40.10      94.872  39      2       0       94      132     20      58      1.28e-19       79.0
1.10.135.10     3.30.590.10     94.737  19      1       0       14      32      1       19      6.20e-05       42.7
1.10.150.20     1.10.150.810    42.222  45      24      1       1       45      10      52      1.12e-04       38.5
1.10.150.850    1.10.150.310    41.270  63      32      2       44      106     1       58      4.15e-07       47.8
1.10.1500.10    3.40.710.10     49.640  139     69      1       1       138     58      196     1.55e-40       139
1.10.287.40     3.30.930.10     83.333  30      4       1       68      96      1       30      3.31e-07       50.4
1.20.120.230    1.20.1420.10    80.769  26      5       0       1       26      138     163     1.26e-06       47.4
1.20.1280.50    3.80.10.10      45.238  42      23      0       1       42      8       49      3.87e-06       43.5
1.20.5.110      1.20.5.170      48.148  54      28      0       10      63      9       62      6.76e-08       46.6
1.20.5.110      1.20.58.70      64.151  53      19      0       13      65      170     222     1.12e-15       72.0
1.20.5.170      1.20.5.110      49.123  57      29      0       7       63      8       64      3.91e-10       52.4
1.20.5.170      2.60.210.10     95.455  22      1       0       45      66      1       22      2.61e-06       44.7
1.20.5.4010     3.30.1370.10    98.305  59      1       0       2       60      1       59      2.24e-35       119
1.20.5.460      3.20.20.220     96.154  26      1       0       1       26      77      102     1.68e-09       52.0
1.20.58.1880    1.10.10.60      48.718  39      20      0       75      113     1       39      2.09e-05       42.0
2.10.310.10     2.30.39.10      75.000  24      6       0       10      33      103     126     1.39e-07       45.8
2.10.310.10     3.30.497.10     54.839  31      14      0       7       37      244     274     1.64e-05       41.6
2.130.10.130    3.40.50.410     43.137  51      29      0       4       54      157     207     6.46e-04       43.1
2.170.300.10    2.10.25.10      57.317  82      32      2       9       90      2       80      8.80e-23       86.7
2.20.70.10      3.90.1750.10    65.625  32      11      0       11      42      1       32      2.96e-09       52.0
2.30.30.360     3.40.850.10     93.478  46      3       0       6       51      1       46      3.11e-23       90.9
2.60.120.10     2.60.120.650    43.182  44      25      0       190     233     130     173     2.31e-04       44.3
2.60.120.650    2.60.120.10     42.045  88      46      1       158     240     167     254     1.92e-15       76.6
3.20.20.140     2.30.40.10      93.333  60      4       0       376     435     49      108     8.54e-32       120
3.30.1120.10    3.40.720.10     88.235  34      4       0       1       34      355     388     1.99e-15       72.0
3.30.30.30      3.30.420.40     97.674  43      1       0       1       43      67      109     1.63e-23       89.0
3.30.40.10      2.60.120.650    46.341  41      20      2       20      59      21      60      8.46e-04       37.7
3.30.60.20      3.30.200.20     42.857  42      24      0       7       48      2       43      6.15e-09       51.2
3.30.70.1400    3.30.1360.120   94.444  18      1       0       69      86      47      64      8.69e-04       39.3
3.30.70.1400    4.10.1250.10    71.429  28      8       0       85      112     10      37      1.02e-05       42.4
3.30.70.1470    3.40.50.1460    46.154  39      21      0       11      49      195     233     5.97e-05       42.7
3.30.70.2820    2.40.50.730     57.143  35      15      0       1       35      49      83      5.05e-06       43.9
3.30.980.10     3.30.54.20      54.545  44      20      0       80      123     13      56      7.46e-10       54.7
3.30.980.10     3.30.930.10     57.143  35      15      0       127     161     2       36      7.54e-05       44.3
3.50.50.80      3.40.50.720     59.184  49      20      0       11      59      17      65      2.59e-12       64.7
3.80.20.20      2.10.220.10     88.000  25      3       0       173     197     1       25      2.00e-08       53.9
3.90.1260.10    1.20.5.470      59.259  27      11      0       198     224     5       31      5.68e-04       39.3
3.90.215.10     4.10.530.10     95.556  90      4       0       158     247     1       90      5.33e-60       187
3.90.70.10      1.10.287.2250   46.667  60      28      3       19      75      10      68      4.64e-05       43.9
3.90.70.10      2.40.50.170     65.000  40      11      1       281     320     3       39      1.62e-08       52.8
4.10.140.10     2.40.10.10      68.293  41      13      0       24      64      7       47      1.72e-12       61.2
4.10.400.10     4.10.1220.10    75.000  20      5       0       18      37      3       22      5.13e-04       35.0
4.10.530.10     3.90.215.10     42.857  49      23      1       1       44      171     219     4.02e-05       42.0
``` 


