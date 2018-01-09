---
layout: post
title: AES ECB Pattern Vunerability
date: 2017-12-31 04:00:00.000
tags: code cryptography
author: Cameron A. Craig
cover: assets/images/2017-12-31-aes-image-patterns/bruges.jpg
published: false
---

An investigation into a pitfall of the AES ECB cipher and how this is overcome in AES CBC.

<a name="contents"></a>
# Contents

<ol>
<li><a href="#title-image">Title Image</a></li>
<li><a href="#introduction">Introduction</a></li>
<li><a href="#purpose">Purpose</a></li>
<li><a href="#background">Background</a></li>
<ol type="i">
	<li><a href="#aes-spec">AES Specification</a></li>
	<li><a href="#ppm-image-format">PPM Image Format</a></li>
</ol>
<li><a href="#method">Method</a></li>
<ol type="i">
	<li><a href="#openssl">OpenSSL</a></li>
	<li><a href="#af_alg">AF_ALG</a></li>
</ol>
<li><a href="#results">Results</a></li>
<li><a href="#conclusion">Conclusion</a></li>
</ol>

<a name="title-image"></a>
# Title Image

The title image of this post is a photo of Bruges, the capital city of Belgium.
Rijndael (now known as AES) was developed by
Vincent *Rij*men and Joan *Dae*men, both from Belgium.
This image is licensed under
Creative Commons Attribution-Share Alike 3.0 Unported, and originated from
[Elke Wetzig](https://commons.wikimedia.org/wiki/User:Elya) on
[Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Bruegge_huidenvettersplein.jpg).

<a name="introduction"></a>
# Introduction

This article discusses the AES ECB pattern vunerability by worked examples using
OpenSSL and AF_ALG.
CBC and ECB modes are different in the way blocks feed into the next block.
AES ECB is the simpler of the two AES variants under comparison,
this runs the risk of people using ECB mode inapproprately.
The inappropriate use of ECB runs the risk of making the ciphertext vunerable
to attack.

The pattern vunerability is illustarted by incrypting PPM images,
allowing the plaintext and ciphertext images to be visualed side-by-side.

All code used in this article is licensed under GNU GPL v2 unless stated
otherwise.
Please get in touch if you find this article useful.

<a name="purpose"></a>
# Purpose

I hope that many people that haven't yet stumbled across AES encryption modes
will find this article interesting.
This article may also consolidate existing knowledge of AES for those that have
come across it before but not been presented with actual working examples.
Although this article uses PPM images as an encryption medium, the patterning
vunerability can also exist (in a less obvious form) in real-world digital
formats such as MPEG-2 and JPEG.

<a name="background"></a>
# Background

<a name="aes-spec"></a>
## AES Specification

[The Advanced Encryption Standard (AES)](https://csrc.nist.gov/csrc/media/publications/fips/197/final/documents/fips-197.pdf)
describes an algorithm that takes in a key and a plaintext as inputs,
and a ciphertext as the output.
This algorithm takes a fixed-size plaintext of 16 bytes, and outputs ciphertext
of the same fixed size.
The key length can be any of the following values: 128 bits, 192 bits, 256 bits
(it's common for key sizes to be specifies in bits rather than bytes).
This same algorithm is used for each mode of encryption.
The difference in each mode is the way each block is connected to the next.

### How the blocks connect

The following diagrams illustrate the two AES modes used:

#### ECB (Electronic Code Book)

![ECB Block Diagram](assets/images/2017-12-31-aes-image-patterns/ecb_block_diagram.png)

#### CBC (Cipher Block Chaining)

![CBC Block Diagram](assets/images/2017-12-31-aes-image-patterns/cbc_block_diagram.png)

### ECB Pattern Vunerability

The reason an image can still be recognised after ECB encryption is because
16 bytes of the same plaintext will always become the same ciphertext,
no matter the position with respect to the rest of the blocks.
So a group of white pixels next to some blue pixels will looks the same in the
ciphertext domain as a group of white pixels next to some pink pixels.

The CBC algorithm defeats this vunerability by XORing the ciphertext of the
previous block with the key, creating dependencies between neighbouring blocks.
Now white pixels will be encrypted to a different colour depending on the
pixel before that, and before that.

<a name="ppm-image-format"></a>
## PPM Image Format

For this article we use the PPM image format, in the binary form.
This format works well for these examples, as the image data section is valid
no matter how messed up the data is.
The PPM format is also extremely simple.
We don't encrpyt the header as this contains crucial image size data that we
need to keep the same.
When we use the PPM format in ASCII form, the encrypted image becomes invalid
as the ciphertext may not be a valid ASCII value.

Example snippet of a PPM image:

```
P6
2048 1536
255
�������������...
...
```

```
P6   --> The magic number identifying the image format
2048 --> Width of image in pixels
1536 --> Height of image in pixels
225  --> Maximum colour value
```
Note that the PPM image format is inefficient by today's standards so won't
be used much. But we don't care about any of that, we're keeping things simple.

More information on the PPM image format can be found by a Google search.


<a name="method"></a>
# Method

<a name="openssl"></a>
## Encrypting PPM images using OpenSSL

The following bash script can be used to encrypt and decrypt an image.

{% highlight bash %}
# Name: ppm-encrypt.sh
# Desc.: Encrypt a PPM image using ECB and CBC AES modes
#        Thanks to:
#           - https://blog.filippo.io/the-ecb-penguin/
#           - https://en.wikipedia.org/wiki/User:Lunkwill
# Author: Cameron A. Craig
# Date: 17/09/2017

# Make sure an argument is given, otherwise print help
if [ $# -eq 0 ]
then
  echo "No image given!"
  exit
fi

# Get image name from first argument


if [ -f $1 ]
then
  IMAGE_PATH=$1
else
  echo "Could not find given image: $1"
  exit
fi

echo "Encrypting image $IMAGE_PATH"

# TODO: Find length of header
# For now assume header length is 3 lines
# perl -ne 'print "$. $_" if m/[\x80-\xFF]/'
HEADER_LINES=3

# Keep the header for the final image
head -n $HEADER_LINES $IMAGE_PATH > header.txt

# Seperate the binary data into its own file
tail -n +$(($HEADER_LINES+1)) $IMAGE_PATH > image.bin

# Encrypt with ECB
openssl enc -aes-128-ecb -nosalt -K 0000000000000000 -in image.bin -out image.ecb.bin
# Join original header and the encrypted image, into a new file
cat header.txt image.ecb.bin > $IMAGE_PATH.ecb.ppm

# Encrypt with CBC
openssl enc -aes-128-cbc -nosalt -pass pass:"PASS" -in image.bin -out image.cbc.bin
# Join original header and the encrypted image, into a new file
cat header.txt image.cbc.bin > $IMAGE_PATH.cbc.ppm


# Remove temporary files
rm header.txt
rm image.bin
rm image.cbc.bin
rm image.ecb.bin


echo "Encrypted image saved to $IMAGE_PATH.ecb.ppm"

{% endhighlight %}

The PPM pixel size is 3 bytes (one byte for each colour, RGB),
and the AES block size is 16 bytes. That is
why areas of the same colour in the plaintext domain appear to have a striped
pattern in the ciphertext domain.

This diagram shows how each pixel of a PPM image is encrypted,
creating a pattern in the encrypted image:

| Pixel # |1 | 1 | 1 | 2| 2 | 2 |3 | 3 | 3 | 4 | 4 | 4 | 5 | 5 | 5 | 6 |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Colour | r | g | b | r | g | b | r | g | b | r | g | b | r | g | b | r |
| Byte Count | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |

Each cell represents a byte. Five full pixels can fit into a block,
with the last pixel overlapping into the next block.

<a name="results"></a>
# Results

The two sets of images shown below are low resulution and are limited in
colour space. This makes the patterns in teh ECB encrypted images very obvious.

## Results of OpenSSL encryption of contiguous data

<main class="flex flex-wrap justify-around align-item items-center">
<div class="flex flex-column items-center">
  <label class="www-example-label bold mb3">Original</label>
  <div class="ampstart-input inline-block relative m0 p0 mb3 ">
  <figure class="ampstart-image-with-caption m0 relative mb4">
  <amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/secret_message.png"></amp-img>
  <figcaption class="h5 mt1 px3">
  Original secret_message.png
  </figcaption>
  </figure>
  </div>
</div>

<div class="flex flex-column items-center">
  <label class="www-example-label bold mb3">CBC</label>
  <div class="ampstart-input inline-block relative m0 p0 mb3 ">
  <figure class="ampstart-image-with-caption m0 relative mb4">
  <amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/secret_message.ppm.cbc.png"></amp-img>
  <figcaption class="h5 mt1 px3">
  secret_message.png encrypted using AES CBC
  </figcaption>
  </figure>
  </div>
</div>

<div class="flex flex-column items-center">
  <label class="www-example-label bold mb3">ECB</label>
  <div class="ampstart-input inline-block relative m0 p0 mb3 ">
  <figure class="ampstart-image-with-caption m0 relative mb4">
  <amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/secret_message.ppm.ecb.png"></amp-img>
  <figcaption class="h5 mt1 px3">
  secret_message.png encrypted using AES ECB
  </figcaption>
  </figure>
  </div>
</div>
</main>







<main class="flex flex-wrap justify-around align-item items-center">
<div class="flex flex-column items-center">
  <label class="www-example-label bold mb3">Original</label>
  <div class="ampstart-input inline-block relative m0 p0 mb3 ">
  <figure class="ampstart-image-with-caption m0 relative mb4">
  <amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/Tux.png"></amp-img>
  <figcaption class="h5 mt1 px3">
  Original Tux.png
  </figcaption>
  </figure>
  </div>
</div>

<div class="flex flex-column items-center">
  <label class="www-example-label bold mb3">CBC</label>
  <div class="ampstart-input inline-block relative m0 p0 mb3 ">
  <figure class="ampstart-image-with-caption m0 relative mb4">
  <amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/Tux.ppm.cbc.png"></amp-img>
  <figcaption class="h5 mt1 px3">
  Tux.png encrypted using AES CBC
  </figcaption>
  </figure>
  </div>
</div>

<div class="flex flex-column items-center">
  <label class="www-example-label bold mb3">ECB</label>
  <div class="ampstart-input inline-block relative m0 p0 mb3 ">
  <figure class="ampstart-image-with-caption m0 relative mb4">
  <amp-img width="128" height="162" layout="responsive" src="assets/images/2017-12-31-aes-image-patterns/Tux.ppm.ecb.png"></amp-img>
  <figcaption class="h5 mt1 px3">
  Tux.png encrypted using AES ECB
  </figcaption>
  </figure>
  </div>
</div>
</main>


<a name="af_alg"></a>
## Encypting PPM Images using AF_ALG

For this example to work you must be using Linux with AF_ALG enabled.
You can check that the `CRYPTO_USER_API` config value has been set to `y` or `m`
by running:

```
cat /boot/config*  | grep CRYPTO_USER_API

```

The following C code acesses the kernel crypto API using the AF_ALG userspace API.
This API has been selected beciause it it particularly suitable to the task at hand.
We can make use of AF_ALG's ability to encrypt non-contiguous data to our advnatage.
Because each pixel in the PPM binary format is 3 bytes, and each AES block is
16 bytes, we create a buffer in a non-contiguous memory location
(a null padding buffer). Each AES block looks at three bytes from the image data
and 13 bytes of the null padding buffer.

We could also move the image data around and re-construct the image after
ciphering, but I thought this was quite an elegant soultion. I think this is
also a good example of where the AF_ALG interface seperates itself from others
such as cryptodev of libtomcrypt. The scatter-gather (non-contiguous) data
inputs and outputs is a powerful feature.

```
```

In order to remove the patterning effect of misaligned pixels relative to the
AES blocks, we can encrypt each pixel in a seperate block, and pad the remainder
of the block with zeroes. This ciphering technique is visualised in the table below:

| Pixel # |1 | 1 | 1 | | | | | | | | | | | | | |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Colour/0 | r | g | b | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Byte Count | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |

By encrypting each pixel in a seperate AES block, we remove the paterning effect.
This results in each pixel colour mapping to the same pixel colour in the ciphertext domain.

To acheive our one-pixel-per-block encryption we use null padding.
This fills the remainder of the 16-byte block with zeros.

### Resulting images

<a name="conclusion"></a>
# Conclusion

I hope this article has provided the reader with an understanding of the AES ECB
pattern vunerability and also provided example usage of a number of software
tools associated with encryption on Linux.

The reader should now have some backgound knowledge that may come in handy when
designing secure systems.
