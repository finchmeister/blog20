---
layout: post
title: Git Commit Hash Number Theory
description: How rare is an all digit commit hash?
summary: How rare is an all digit commit hash?
tags: [tech]
---


Life as a programmer is often mundane. You start with a brief to make something happen on a screen. Spend hours upon hours sat at a desk, typing and tapping away, before eventually finding out the feature you've just lovingly crafted isn't needed anymore. It means that small, out of the ordinary events in the software development process can catch one's curiosity and lead to unexpected tangents in the pursuit of knowledge, such as 'how rare is this _all digit_ git commit hash I have just generated?'

![Short commit]({{ '/assets/git-number-theory/shortgitcommit.png' | relative_url }})
![Full commit]({{ '/assets/git-number-theory/gitcommit.png' | relative_url }})

I was sure I had seen these things before, but I had no clue as to the significance of it, so I went on a journey to find out.

At a high level, a git commit hash is a SHA1 hash of the state of the git repository at the time of the commit. A short git commit hash is an abbreviation of the hash to the first 7 characters, it is almost certainly unique within a repository and git will increase the number of characters used if it is not.

To empirically find out the probability of an all digit hash, I wrote a simulation that generated a lot of SHA1 hashes and returned the percentage of the total which start with 7 digits.

It takes just a single character in the hash subject to change for the hash output to be vastly different, (called the _avalanche effect_), so for ease I simply increment a counter (known as the _nonce_) to generate a new hash subject. 

As an aside, Bitcoin does something similar in its proof of work algorithm to generate blocks and prevent tampering to the blockchain. It relies on incrementing a counter and generating a hash based upon the transaction log until the resulting hash starts with a certain number of 0s (depending on the network difficulty). It is very easy to verify the hash but takes a lot of computational power to generate a valid one in the first place. This is known as the [HashCash](https://en.bitcoin.it/wiki/Hashcash) proof of work system.

The PHP script:

~~~php
<?php

function getShortHash(string $subject): string
{
    return substr(sha1($subject), 0, 7);
}

function isAllInt(string $subject): bool
{
    return preg_match('/^\d{7}$/', $subject) === 1;
}

$n = 10000;
$allInt = $notAllInt = 0;
$randomBytes = random_bytes(10);
for ($i = 0; $i < $n; $i++) {
    $subject = $randomBytes . $i;
    $shortHash = getShortHash($subject);
    if (isAllInt($shortHash)) {
        $allInt++;
    } else {
        $notAllInt++;
    }
}
$ratio = $allInt/$n * 100;

echo "All int: {$allInt}; Not all int: {$notAllInt}; All int: {$ratio} %\n";
~~~


The results after running the script 5 times:

~~~bash
All int: 387; Not all int: 9613; All int: 3.87 %
All int: 408; Not all int: 9592; All int: 4.08 %
All int: 356; Not all int: 9644; All int: 3.56 %
All int: 374; Not all int: 9626; All int: 3.74 %
All int: 357; Not all int: 9643; All int: 3.57 %
~~~

We can see between 3.56% and 4.08% of random SHA1 hashes will start with 7 integers - roughly 1 in 27. In reality, all digit short hashes are not that rare.

Let's look a bit deeper as to _why_ the probability of all integer short hashes is around 1 in 27.

A SHA1 hash is represented here as a 40 character hexadecimal string. Each character in hexadecimal has 16 options, 0-9 or a-f. Assuming a SHA1 hash has a seemingly random output where each character is equally likely to be selected upon a hash, then the likelihood of the first character being an integer is 10/16, the second is also 10/16, and so on for the first 7 characters. We can calculate the probably like so: 

```
10/16 * 10/16 * 10/16 * 10/16 * 10/16 * 10/16 * 10/16 
= (10/16)^7
= 10000000/268435456
= 0.03725290298
```

i.e.: 3.725% or 1 in 26.67.

3.725% fits within the range of values computed by the simulation, validating that the theory and the practice tie up.

According to the law of large numbers, as we increase the number of trials, the average of the result will tend towards the true expected value. The mathematics tells us that the expected value is 3.725%, so let's see what happens if we run the script with a much higher number of iterations. 

Upping the iterations of the script by 5000 to 500 million and running this on an oversized Google Cloud Platform VM we get:

```
All int: 18630499; Not all int: 481369501; All int: 3.7260998 %
```

Extremely close to the expected value, just a 0.02952483% difference. This demonstrates that the output of SHA1 algorithm is seemingly random even though we are only incrementing a counter in the hash subject - the _avalanche effect_. This is what we would expect from a hashing algorithm, if there were a correlation between the input and output, the hashing algorithm would be considered broken as it would open up the possibility of reversing a hash via a cryptanalytic attack.


## How about git hashes that are all letters?

Naturally I considered how rare all letter short git hashes were:

```
All letters: 12; Not all letters: 9988; All letters: 0.12 %
All letters: 10; Not all letters: 9990; All letters: 0.1 %
All letters: 14; Not all letters: 9986; All letters: 0.14 %
All letters: 11; Not all letters: 9989; All letters: 0.11 %
All letters: 7; Not all letters: 9993; All letters:: 0.07 %
```

Following the same logic as before but with 6 possible characters (a-f) we get 

```
(6/16)^7 = 0.001042842865 ~ 0.1%. 
```


I.e., 1 in 959 commits.

A much, much rarer event.

## What is the rarest commit hash I could find in the wild?

I took a look at the biggest git repo I could think of, the Linux kernel, which at the time of writing was 2.5GB and had 856,459 commits. I extracted the commit hashes to a file then grepped for the longest starting sequence of letters and digits.

```bash
~/linux$ git log --pretty=oneline | cut -c 1-40 > git_log.log
~/linux$ grep -E "^[0-9]{27,}" git_log.log 
932505776779430053766113965c21cfb7ab823a
287774414568010855642518513f085491644061

~/linux$ grep -E "^[a-z]{15,}" git_log.log 
fdbdfefbabefcdf3f57560163b43fdc4cf95eb2f
```

There are some pretty unusual hashes there, two starting with 27 digits, a 1 in 324,518 chance, and [one](https://github.com/torvalds/linux/commit/fdbdfefbabefcdf3f57560163b43fdc4cf95eb2f) starting with 15 letters, a 1 in 2,452,059 chance. You've got more chances of winning £10,000 every month for one year with the National Lottery - Set For Life lottery [1 in 1,704,377](https://en.wikipedia.org/wiki/National_Lottery_(United_Kingdom)#Set_For_Life) than getting a git commit hash that looks like that.


Who'd have thought the life of a programmer was mundane now?