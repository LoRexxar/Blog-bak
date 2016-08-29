---
title: crypto 简单的RSA
date: 2016-08-29 12:27:53
tags:
- crypto
- RSA
- rabins
categories:
- Blogs
---

前段时间没事做就去玩了玩国外的icectf，虽然没听说过，但是题目还不错，比较新手向，遇到很多有意思的题目，其中就包括很多简单的crypto题目，密码学一直是信安很重要的东西，但是没天赋学不好，无意中接触了下实战，稍微记录下...

<!--more-->

# RSA #

没啥可说的，n,d,e,phi都有，直接python解就可以了

```
直接python解
>>> m = pow(c,d,n)

>>> m
3843655260524402023604596518050334491485822435243281383499136834535067384556161639265107050668678281151778547364113350618891028501821403003350717660361853L

>>> hex(m)
'0x4963654354467b7273615f69735f617765736f6d655f7768656e5f757365645f636f72726563746c795f6275745f686f727269626c655f7768656e5f6e6f747dL'

>>> hex(m)[2:-1].decode('hex')
'IceCTF{rsa_is_awesome_when_used_correctly_but_horrible_when_not}'

```

# RSA2 #

题目比较像正常的RSA题目了，给了n，e，c，首先需要分解n，这里我用的是yafu，别的也没用过，不知道怎么样

分解n得到pq
```
N=0xee290c7a603fc23300eb3f0e5868d056b7deb1af33b5112a6da1edc9612c5eeb4ab07d838a3b4397d8e6b6844065d98543a977ed40ccd8f57ac5bc2daee2dec301aac508f9befc27fae4a2665e82f13b1ddd17d3a0c85740bed8d53eeda665a5fc1bed35fbbcedd4279d04aa747ac1f996f724b14f0228366aeae34305152e1f430221f9594497686c9f49021d833144962c2a53dbb47bdbfd19785ad8da6e7b59be24d34ed201384d3b0f34267df4ba8b53f0f4481f9bd2e26c4a3e95cd1a47f806a1f16b86a9fc5e8a0756898f63f5c9144f51b401ba0dd5ad58fb0e97ebac9a41dc3fb4a378707f7210e64c131bca19bd54e39bbfa0d7a0e7c89d955b1c9f

P8 = 57970027
PRP609 = 518629368090170828331048663550229634444384299751272939077168648935075604180676006392464524953128293842996441022771890719731811852948684950388211907532651941639114462313594608747413310447500790775078081191686616804987790818396104388332734677935684723647108960882771460341293023764117182393730838418468480006985768382115446225422781116531906323045161803441960506496275763429558238732127362521949515590606221409745127192859630468854653290302491063292735496286233738504010613373838035073995140744724948933839238851600638652315655508861728439180988253324943039367876070687033249730660337593825389358874152757864093

```

算的phi=(p-1)(q-1)

这里懵了一下，因为不知道怎么算d，自己实现又跑不出来，问学长得知有库实现

```
>>> from gmpy2 import invert

>>> d = invert(e, phi)

>>> d
mpz(18371016466543300213341861192944643232713350676408895652887982330667640552462739649024950272690814682262459294225948873554583004877005275309848872991260865129018162831677976707577891281555755266551429653910739381919356650113933918645959774213561168816300404880716256672857759491893067485531099657843084148100498000636075953794011472525339245044341605635286960339120021682607930670688995965934146606840559736391472945463866017297966536379449623206354851141502561617045225910476908953680145813578977873197587798615715573389247939388328272149466230224059763007328120238518721502908638644530839386083428373445455039926609L)

```

解出来就是d了，算flag
```
>>> hex(m)[4:-2].decode('hex')
'ceCTF{next_time_check_your_keys_arent_factorable'
```


# RSA? #

e=1,所以密文等于密钥，直接解密

```
N=0x180be86dc898a3c3a710e52b31de460f8f350610bf63e6b2203c08fddad44601d96eb454a34dab7684589bc32b19eb27cffff8c07179e349ddb62898ae896f8c681796052ae1598bd41f35491175c9b60ae2260d0d4ebac05b4b6f2677a7609c2fe6194fe7b63841cec632e3a2f55d0cb09df08eacea34394ad473577dea5131552b0b30efac31c59087bfe603d2b13bed7d14967bfd489157aa01b14b4e1bd08d9b92ec0c319aeb8fedd535c56770aac95247d116d59cae2f99c3b51f43093fd39c10f93830c1ece75ee37e5fcdc5b174052eccadcadeda2f1b3a4a87184041d5c1a6a0b2eeaa3c3a1227bc27e130e67ac397b375ffe7c873e9b1c649812edcd

e=0x1

c=0x4963654354467b66616c6c735f61706172745f736f5f656173696c795f616e645f7265617373656d626c65645f736f5f63727564656c797d

```

```
IceCTF{falls_apart_so_easily_and_reassembled_so_crudely}
```

# Round Rabins! #

之前虽然研究过一点儿一点儿rabin，但是不知道为什么完全算不出来

读了一篇wp，
[https://github.com/WCSC/writeups/tree/master/icectf-2016/Round-Rabins](https://github.com/WCSC/writeups/tree/master/icectf-2016/Round-Rabins)

首先我们从题目中获得的信息有
```
N=0x6b612825bd7972986b4c0ccb8ccb2fbcd25fffbadd57350d713f73b1e51ba9fc4a6ae862475efa3c9fe7dfb4c89b4f92e925ce8e8eb8af1c40c15d2d99ca61fcb018ad92656a738c8ecf95413aa63d1262325ae70530b964437a9f9b03efd90fb1effc5bfd60153abc5c5852f437d748d91935d20626e18cbffa24459d786601

c=0xd9d6345f4f961790abb7830d367bede431f91112d11aabe1ed311c7710f43b9b0d5331f71a1fccbfca71f739ee5be42c16c6b4de2a9cbee1d827878083acc04247c6e678d075520ec727ef047ed55457ba794cf1d650cbed5b12508a65d36e6bf729b2b13feb5ce3409d6116a97abcd3c44f136a5befcb434e934da16808b0b
```

除此之外，我们还知道是rabin加密，公式和原理我这里就不写了，我们知道密文等于明文的平方对N取余，我们用yafu分解N发现N=p2

```
P154 = 8683574289808398551680690596312519188712344019929990563696863014403818356652403139359303583094623893591695801854572600022831462919735839793929311522108161
P154 = 8683574289808398551680690596312519188712344019929990563696863014403818356652403139359303583094623893591695801854572600022831462919735839793929311522108161
```

原谅我数学水平不高，所以还是没弄明白怎么回事，但是还是有脚本

```
#!/usr/bin/env python
'''
Rabin cryptosystem challenge:
N=0x6b612825bd7972986b4c0ccb8ccb2fbcd25fffbadd57350d713f73b1e51ba9fc4a6ae862475efa3c9fe7dfb4c89b4f92e925ce8e8eb8af1c40c15d2d99ca61fcb018ad92656a738c8ecf95413aa63d1262325ae70530b964437a9f9b03efd90fb1effc5bfd60153abc5c5852f437d748d91935d20626e18cbffa24459d786601


c=0xd9d6345f4f961790abb7830d367bede431f91112d11aabe1ed311c7710f43b9b0d5331f71a1fccbfca71f739ee5be42c16c6b4de2a9cbee1d827878083acc04247c6e678d075520ec727ef047ed55457ba794cf1d650cbed5b12508a65d36e6bf729b2b13feb5ce3409d6116a97abcd3c44f136a5befcb434e934da16808b0b
'''
# some functions from http://codereview.stackexchange.com/questions/43210/tonelli-shanks-algorithm-implementation-of-prime-modular-square-root/43267
def legendre_symbol(a, p):
    """
    Legendre symbol
    Define if a is a quadratic residue modulo odd prime
    http://en.wikipedia.org/wiki/Legendre_symbol
    """
    ls = pow(a, (p - 1)/2, p)
    if ls == p - 1:
        return -1
    return ls

def prime_mod_sqrt(a, p):
    """
    Square root modulo prime number
    Solve the equation
        x^2 = a mod p
    and return list of x solution
    http://en.wikipedia.org/wiki/Tonelli-Shanks_algorithm
    """
    a %= p

    # Simple case
    if a == 0:
        return [0]
    if p == 2:
        return [a]

    # Check solution existence on odd prime
    if legendre_symbol(a, p) != 1:
        return []

    # Simple case
    if p % 4 == 3:
        x = pow(a, (p + 1)/4, p)
        return [x, p-x]

    # Factor p-1 on the form q * 2^s (with Q odd)
    q, s = p - 1, 0
    while q % 2 == 0:
        s += 1
        q //= 2

    # Select a z which is a quadratic non resudue modulo p
    z = 1
    while legendre_symbol(z, p) != -1:
        z += 1
    c = pow(z, q, p)

    # Search for a solution
    x = pow(a, (q + 1)/2, p)
    t = pow(a, q, p)
    m = s
    while t != 1:
        # Find the lowest i such that t^(2^i) = 1
        i, e = 0, 2
        for i in xrange(1, m):
            if pow(t, e, p) == 1:
                break
            e *= 2

        # Update next value to iterate
        b = pow(c, 2**(m - i - 1), p)
        x = (x * b) % p
        t = (t * b * b) % p
        c = (b * b) % p
        m = i

    return [x, p-x]

def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('modular inverse does not exist')
    else:
        return x % m


# This finds a solution for c = x^2 (mod p^2)
def find_solution(c, p):
    '''
    Hensel lifting is fairly simple.  In one sense, the idea is to use
    Newton's method to get a better result.  That is, if p is an odd
    prime, and

                            r^2 = n (mod p),

    then you can find the root mod p^2 by changing your first
    "approximation" r to

                        r - (r^2 - n)/(2r) (mod p^2).

    http://mathforum.org/library/drmath/view/70474.html                    
    '''
    n = p ** 2
    # Get square roots for x^2 (mod p)
    r = prime_mod_sqrt(c,p)[0]

    inverse_2_mod_n = modinv(2, n)
    inverse_r_mod_n = modinv(r, n)

    new_r = r - inverse_2_mod_n * (r - c * inverse_r_mod_n)

    return new_r % n

if __name__ == "__main__":
    # These are the given values
    n = 0x6b612825bd7972986b4c0ccb8ccb2fbcd25fffbadd57350d713f73b1e51ba9fc4a6ae862475efa3c9fe7dfb4c89b4f92e925ce8e8eb8af1c40c15d2d99ca61fcb018ad92656a738c8ecf95413aa63d1262325ae70530b964437a9f9b03efd90fb1effc5bfd60153abc5c5852f437d748d91935d20626e18cbffa24459d786601L
    # n is a perfect square: n = p * p
    p = 0xa5cc6d4e9f6a893c148c6993e1956968c93d9609ed70d8366e3bdf300b78d712e79c5425ffd8d480afcefc71b50d85e0914609af240c981c438acd1dcb27b301L
    # encrypted message
    c = 0xd9d6345f4f961790abb7830d367bede431f91112d11aabe1ed311c7710f43b9b0d5331f71a1fccbfca71f739ee5be42c16c6b4de2a9cbee1d827878083acc04247c6e678d075520ec727ef047ed55457ba794cf1d650cbed5b12508a65d36e6bf729b2b13feb5ce3409d6116a97abcd3c44f136a5befcb434e934da16808b0bL

    solution = find_solution(c, p)
    print hex(solution)[2:-1].decode("hex")
```


```
IceCTF{john_needs_to_get_his_stuff_together_and_do_things_correctly}
```