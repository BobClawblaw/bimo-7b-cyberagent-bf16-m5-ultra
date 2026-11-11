| Check | Passed | Score | Time | Note |
|---|---:|---:|---:|---|
| drafts | 12/12 | 100.0% | 50s | drafted 52 tok/s vs undrafted 47 tok/s |
| streaming | 4/4 | 100.0% | 17s | identical |
| multiturn | 8/8 | 100.0% | 7s | recalled 8/8 facts |
| stop | 3/3 | 100.0% | 2s | 3/3 stopped before the string |
| thinking | 0/10 | 0.0% | 116s | 10 replies without reasoning_content, 0 with think tags in content |
| copy | 2/2 | 100.0% | 4s | 2/2 byte-identical |
| max-tokens | 2/3 | 66.7% | 1s | 2/3 exact |
| tools | 9/60 | 15.0% | 35s | "What's the weather in Lisbon?": got [] |
| json | 25/30 | 83.3% | 10s | #6: Expecting ',' delimiter: line 1 column 102 (char 101) |
| ifeval | 61/150 | 40.7% | 864s | loose 61/150, strict 57/150, 150 covered prompts |
| gsm8k | 180/250 | 72.0% | 832s | misses at [2, 4, 7, 8, 9, 12, 14, 20, 21, 24] |
| mgsm | 96/240 | 40.0% | 1002s | de 13/30, es 17/30, fr 18/30, ja 10/30, zh 17/30, ru 14/30, sw 3/30, bn 4/30 |
| mmlu | 182/342 | 53.2% | 634s | 57 subjects x 6; weakest machine_learning 0/6, moral_scenarios 0/6, high_school_us_history 1/6 |
| math500 | 207/500 | 41.4% | 7425s | L1 33/43 L2 54/90 L3 50/105 L4 50/128 L5 20/134, 105 without a boxed answer |
| gpqa | 47/198 | 23.7% | 2315s | Biology 9/19, Chemistry 21/93, Physics 17/86, 31 without an answer |
| humaneval | 0/0 | 0.0% | 0s | failed: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issue |
| repetition | 0/8 | 0.0% | 80s | repeated 4-grams 36.93% mean, max 85.56%; 4 replies hit max_tokens |
| long-context | 0/0 | 0.0% | 25s | failed: HTTP Error 400: Bad Request |
| long-tasks | 0/52 | 0.0% | 25s | mk-niah 0/1@28k, multi-value 0/4@28k, var-track 0/5@28k, common-words 0/10@32k, mk-niah 0/1@128k, multi-value 0/4@128k, var-track 0/5@128k, common-words 0/10@128k, mk-niah 0/1@512k, var-track 0/5@512k, mk-niah 0/1@900k, var-track 0/5@900k |
| soak | 60/60 | 100.0% | 252s | 3 chats x 20 turns at once beside a decoding stream (5.2k-5.8k-token prompts): 60/60 equal to the same request alone, 0 errors, server answers after; 60/60 concurrent turns resumed a kept state |
| snapshot | 12/12 | 100.0% | 34s | replies written to suite/replies.json |
| vision | 0/6 | 0.0% | 1s | shapes: missing ['circle', 'red', 'square/rectangle', 'blue', '42'] in '<error: HTTP Error 400: Bad Request>' |
| docvision | 0/0 | 0.0% | 1s | failed: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issue |
| concurrent | 12/12 | 100.0% | 50s | 16 streams together vs alone |
| vision-concurrent | 0/0 | 0.0% | 0s | failed: every case refused alone: shapes, text, count, two images, video motion, video colours |
