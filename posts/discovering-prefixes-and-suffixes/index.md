---
title: Discovering Prefixes and Suffixes
date: 2012-09-25
published: true
author: gaoyuan
image: img/fig.png
mathjax: on
tags: language, text processing, morphology, coding, unix, shell script
---
English is a language that has a concatinative morphology. For example, the word *preinstallation* can be split into three parts -- a prefix *pre*, a stem *install* and a suffix *ation*. Given a corpus of english words, how do we discover the prefixes and suffixes computationally? This is an interesting question, for we can use similar methods to analyze other languages that also share a concatinative morphology.

The corpus of english words can be represented by a forward tree (and a backward tree). The root of the forward tree is the token that denotes the start of word. Each node in the tree corresponds to a character. A word in the corpus is therefore a path starting from the root. Similarly, for the backward tree, the root is the end of word token. A path starting from the root corresponds to a word reading backwards.

After building this tree representation, we can calculate the frequency of each node, which is defined as the frequency of the path starting from the root to that node. Since prefixes (or suffixes) tend to appear more often, high frequency candidates are more likely to be considered prefixes (or suffixes) given a fixed character sequence length, or equivalently, a fixed depth in the tree. For instance, for a forward tree, we sort the candidates with tree depth 5 (word length 4) by their frequencies, we get 

>
|  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | &nbsp; |
| ---- | --- |
| OVER | 378 |
| INTE | 338 |
| COMP | 233 |
| CONS | 212 |
| UNDE | 205 |
| TRAN | 183 |
| CONT | 179 |
| STRA | 150 |
| COMM | 142 |
| PRES | 133 |
| ... | ... |

It seems that we can capture some prefixes solely by their frequencies. The guy on the top of the list -- *OVER* is a common prefix. However, a lot of the high frequency stuff are not actually prefixes, like *INTE* and *UNDE*. Why do they have a high frequency though? That is because *INTER* and *UNDER* are prefixes. So high frequency itself is not a sufficient criterion for a prefix.

It turns out we can eliminate the bad ones by the distribution of their children. For all the characters following *INTE*, most of their occurancies should be *R*, therefore the distribution of the children of *INTE* should be highly biased. A measure that people usually use to discribe this is called **Entropy**. It is defined as $-\sum_ip_i \ln p_i$, where $p_i$ is the probability of the i-th child. A high entropy implies the children are evenly distributed. Node with a high entropy is evidence for a good cut between prefix and stem. That is because varies stems can be attached to the same prefix, so the character following the prefix is rather chaotic.

It turns out there exists some cases where the boundary of a prefix or suffix does not have a high entropy. The suffix *-ing* is a good example. Unfortunately, *ling* appears more often than other *-ing*'s, like *ting* or *ming*. Therefore if you look at *ing*, it doesn't have a very high entropy. Combining the frequency and entropy criterions helps a lot, but there are still some corner cases.

What if we print the frequency and entropy path of each candidate? Below we selected some of the top depth-6 candidates that have both high frequencies and high entropies. We print the frequency and entropy path of each candidate:

>
| candidate&nbsp;&nbsp; | freq&nbsp;&nbsp; | entropy |
| --------- | -- | ------- |
| I N T E R | 275 | 2.71212 |
| I N T E	| 338 | 0.745482 |
| I N T	| 439 | 0.818788 |
| I N	| 1663 | 2.41109 |
| I	| 2849 | 1.64657 |
| M I C R O | 87 | 2.63535 |
| M I C  R | 87 | 0 |
| M I C	| 179 | 1.33012 |
| M I	| 1377 | 2.23578 |
| M	| 8424 | 1.7602 |
| U N D E R | 187 | 2.67801 |
| U N D E	| 205 | 0.466284 |
| U N D	| 247 | 0.711374 |
| U N	| 1109 | 2.68328 |
| U	| 1608 | 1.33352 |
| W A T E R | 51 | 2.52729 |
| W A T E	| 51 | 0 |
| W A T	| 99 | 1.67263 |
| W A	| 854 | 2.45031 |
| W	| 3543 | 1.7672 |

The feature becomes clear -- a prefix should be one such that its parent has a low entropy and itself has a high entropy. The same applies to suffixes. Now if we look at the entropy path of *ing*, we find

>
| candidate&nbsp;&nbsp; | entropy |
| --------- | --- |
| ING | 2.66174 |
| NG | 0.282628 |
| G | 0.646555 |

Since *NG* has a low entropy and *ING* has a high entropy, it becomes clear that *ING* should be a suffix.

The experiment above uses the corpus [CMU Pronouncing Dictionary](http://www.speech.cs.cmu.edu/cgi-bin/cmudict). Below is the shell script to generate the frequencies and entropies for the forward and backward tree. It uses *ngram-count*, the tool from [SRILM](http://www.speech.sri.com/projects/srilm/download.html) that generates all the ngrams based on the training text.

```bash
\#!/bin/sh
\#--Generate frequency and entropy for forward&backward tree.--
\# == parameters ==
DICT=../data/cmudict.0.7a.txt
ORDER=7
\# == function ngramFrequencyEntropy ==
\# Generate 1-$2 order grams with frequency and entropy
\# $1 : file name
\# $2 : order
ngramFrequencyEntropy(){
  CMD="ngram-count -text $1 -order $2"
  for (( i = 1; i <= $2; i ++ )); do
    CMD=$CMD" -write$i $1.ORDER$i"
  done
  eval $CMD
  for (( i = 1; i <= $2; i ++ )); do
    grep '^\<s\>' $1.ORDER$i > $1.ORDER$i.PRE
  done  
  eval "cat $1.ORDER[1-$(($ORDER-1))].PRE > $3" \# cat all the prefixes
  \# add the entropy of each node
  LINENUM=1
  AWKCMD='{sum+=$NF; a[++i]=$NF } END ' \# simply because its too long...
  AWKCMD+='{ for(j=1;j<=i;j++) {p = a[j]/sum; entropy += -p\*log(p)} } END '
  AWKCMD+='{print entropy}'
  for (( i = 1; i < $2; i ++ )); do
    while IFS= read -r line; do 
      ENTROPY=`echo "$line" | rev | cut -f2- | rev | \
      grep -f- $1.ORDER$(($i+1)).PRE | awk "$AWKCMD"`
      if [ "x$ENTROPY" = "x" ]; then
        ENTROPY=0
      fi
      awk -v ent=$ENTROPY -v ln=$LINENUM \
      '{if (NR==ln) {print $0, ent} else {print $0}}' $3 > TEMP && mv TEMP $3
      LINENUM=$(($LINENUM+1))
    done < $1.ORDER$i.PRE
  done 
}
\# == Generate phoneme and grapheme suffix and prefix trees ==
echo 'Generating grapheme trees ...'
sed '/\;\;\;/d;s/  .\*//g;/[^A-Za-z]/d' $DICT | \
uniq | sed 's/./ &/g;s/^ //' > CH \# char-only
ngramFrequencyEntropy CH $ORDER ch.order$ORDER
rev CH > CH_REV
ngramFrequencyEntropy CH_REV $ORDER ch_rev.order$ORDER
echo 'Generating phoneme trees ...'
sed '/\;\;\;/d;s/.\*  //g' $DICT > PH \# phone-only
ngramFrequencyEntropy PH $ORDER ph.order$ORDER
rev PH > PH_REV
ngramFrequencyEntropy PH_REV $ORDER ph_rev.order$ORDER
\# == Remove temp files ==
rm CH CH_REV PH PH_REV \*ORDER\*
```

Below is the code that generates possible candidates for prefixes and suffixes by selecting those that have both high frequencies and high entropies:

```bash
\#!/bin/sh
\# -- generate the candidates using frequency and entropy --
\# == parameters ==
ORDER=7
FREQ_CUTOFF=50
ENTR_CUTOFF=50
topFreqEntr(){
  for (( i = 3; i < $2; i ++ )); do
    cat $1.order$i | sort -k$((i+1)) -r -n | head -$FREQ_CUTOFF > topfreq
    cat $1.order$i | sort -k$((i+2)) -r -n | head -$ENTR_CUTOFF > topentr
    sort -o topfreq topfreq
    sort -o topentr topentr
    comm -12 topfreq topentr | sed "s/^\t//g" \> $1.order$i.candidates
  done
}
topFreqEntr ch $ORDER
topFreqEntr ph $ORDER
topFreqEntr ch_rev $ORDER
topFreqEntr ph_rev $ORDER
\# == clear junk files == 
rm topfreq topentr
```

Below is the code that prints the frequency and entropy path of candidates:

```bash
\#!/bin/sh
\# -- print the path of the candidates --
\# == parameters ==
ORDER=7
printPath(){
  while read line; do
    NUM_COL=`expr $2 - 1`
    echo $line
    while [ $NUM_COL -ge 2 ]; do
      echo $line | cut -d' ' -f 1-$NUM_COL | grep -f- $1.order$NUM_COL
      NUM_COL=`expr $NUM_COL - 1`
    done
    echo ""
  done < $1.order$2.candidates
}
for (( i = 3; i < $ORDER; i++ )); do
  printPath ch $i > ch.order$i.candidates.path
  printPath ph $i > ph.order$i.candidates.path
  printPath ch_rev $i > ch_rev.order$i.candidates.path
  printPath ph_rev $i > ph_rev.order$i.candidates.path
done
```
