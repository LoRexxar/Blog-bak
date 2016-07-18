title: sctfq1_Obfusion_writeup
date: 2016-04-11 13:25:25
tags:
- Blogs
- ctf
categories:
- Blogs
---
昨天打了一场叫做sctf q1的外国比赛，反正是一大堆英语，注册的时候也没太理解怎么回事，好像是面向高中生的ctf，不管怎么说，高分的题目还是有一些质量，这里就留下web5 obfustion的wp.

<!--more-->
首先题目是一道js的逻辑反混淆，这种题还是老做法，先拖进控制台一步步分析。

首先源码是这样的
```
var _ = { 0x4c19cff: "random", 0x4728122: "charCodeAt", 0x2138878: "substring", 0x3ca9c7b: "toString", 0x574030a: "eval", 0x270aba9: "indexOf", 0x221201f: function(_9) { var _8 = []; for (var _a = 0, _b = _9.length; _a < _b; _a++) { _8.push(Number(_9.charCodeAt(_a)).toString(16)); } return "0x" + _8.join(""); }, 0x240cb06: function(_2, _3) { var _4 = Math.max(_2.length, _3.length); var _7 = _2 + _3; var _6 = ""; for(var _5=0; _5<_4; _5++) { _6 += _7.charAt((_2.charCodeAt(_5%_2.length) ^ _3.charCodeAt(_5%_3.length)) % _4); } return _6; }, 0x5c623d0: function(_c, _d) { var _e = ""; for(var _f=0; _f<_d; _f++) { _e += _c; } return _e; } };
			var $ = [ 0x4c19cff, 0x3cfbd6c, 0xb3f970, 0x4b9257a, 0x1409cc7, 0x46e990e, 0x2138878, 0x1e1049, 0x164a1f9, 0x494c61f, 0x490f545, 0x51ecfcb, 0x4c7911a, 0x29f7b65, 0x4dde0e4, 0x49f889f, 0x5ebd02c, 0x556f342, 0x3f7f3f6, 0x11544aa, 0x53ed47d, 0x381f2118, 0x2e9d65d, 0x5c623d0, 0x32e8f8b, 0x3ca9c7b, 0x367a49b, 0x360179b, 0x5c862d6, 0x30dc1af, 0x7797d1, 0x221201f, 0x5eb4345, 0x5e9baad, 0x39b3b47, 0x32f0b8f, 0x48554de, 0x3e8b5e8, 0x5e4f31f, 0x48a53a6, 0x270aba9, 0x240cb06, 0x574030a, 0x1618f3a, 0x271259f, 0x3a306e5, 0x1d33b46, 0x17c29b5, 0x1cf02f4, 0xeb896b ];
			var a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r, s, t, u, v, w, x, y, z;
			function check() {
				var answer = document.getElementById("message").value;
				var correct = (function() {
					try {
						h = new MersenneTwister(parseInt(btoa(answer[_[$[6]]](0, 4)), 32));
						e = h[_[$[""+ +[]]]]()*(""+{})[_[0x4728122]](0xc); for(var _1=0; _1<h.mti; _1++) { e ^= h.mt[_1]; }
						l = new MersenneTwister(e);
						l.random(); l.random(); l.random();
						o = answer.split("_");
						i = l.mt[~~(h.random()*$[0x1f])%0xff];
						s = ["0x" + i[_[$[$.length/2]]](0x10), "0x" + e[_[$[$.length/2]]](0o20).split("-")[1]];
						e =- (this[_[$[42]]](_[$[31]](o[1])) ^ s[0]); if (-e != $[21]) return false;
						e ^= (this[_[$[42]]](_[$[31]](o[2])) ^ s[1]); if (-e != $[22]) return false; e -= 0x352c4a9b;
						t = new MersenneTwister(Math.sqrt(-e));
						h.random();
						a = l.random();
						t.random();
						y = [ 0xb3f970, 0x4b9257a, 0x46e990e ].map(function(i) { return $[_[$[40]]](i)+ +1+ -1- +1; });
						o[0] = o[0].substring(5); o[3] = o[3].substring(0, o[3].length - 1);
						u = ~~~~~~~~~~~~~~~~(a * i);
						a = parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(_[$[31]](o[0]));
						r = (h.random() * l.random() * t.random()) / (h.random() * l.random() * t.random());
						e ^= ~r;
						r = (h.random() / l.random() / t.random()) / (h.random() * l.random() * t.random());
						e ^= ~~r;
						a += _[$[31]](o[3].substring(o[3].length - 2)).split("x")[1];
						d = parseInt(a, 16) == (Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ "");
						i = 0xff;
						n = (f = _[$[23]](o[3].charAt(o[3].length - 4), 3)) == o[3].substring(1, 4);
						g = 111;
						t = _[$[23]](o[3].charAt(3), 3) == o[3].substring(5, 8) && (o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab;
						h = ((g ^ e ^ 96) & i).toString(16);
						i = o[3].split(f).join("");
						s = i.substring(0, 2) == h;
						return (n & t & s) === 1 || (n & t & s) === true;
					} catch (e) {
						console.log("screw you");
						return false;
					}
				})();

				document.getElementById("message").placeholder = correct ? "correct" : "wrong";
				if (correct) {
					document.getElementById("message").disabled = true;
				} else {
					document.getElementById("message").value = "";
				}
			};
```
分析花了很长时间...

```
var _ = { 0x4c19cff: "random", 0x4728122: "charCodeAt", 0x2138878: "substring", 0x3ca9c7b: "toString", 0x574030a: "eval", 0x270aba9: "indexOf", 0x221201f: function(_9) { var _8 = []; for (var _a = 0, _b = _9.length; _a < _b; _a++) { _8.push(Number(_9.charCodeAt(_a)).toString(16)); } return "0x" + _8.join(""); }, 0x240cb06: function(_2, _3) { var _4 = Math.max(_2.length, _3.length); var _7 = _2 + _3; var _6 = ""; for(var _5=0; _5<_4; _5++) { _6 += _7.charAt((_2.charCodeAt(_5%_2.length) ^ _3.charCodeAt(_5%_3.length)) % _4); } return _6; }, 0x5c623d0: function(_c, _d) { var _e = ""; for(var _f=0; _f<_d; _f++) { _e += _c; } return _e; } };
console.log(_)
var $ = [ 0x4c19cff, 0x3cfbd6c, 0xb3f970, 0x4b9257a, 0x1409cc7, 0x46e990e, 0x2138878, 0x1e1049, 0x164a1f9, 0x494c61f, 0x490f545, 0x51ecfcb, 0x4c7911a, 0x29f7b65, 0x4dde0e4, 0x49f889f, 0x5ebd02c, 0x556f342, 0x3f7f3f6, 0x11544aa, 0x53ed47d, 0x381f2118, 0x2e9d65d, 0x5c623d0, 0x32e8f8b, 0x3ca9c7b, 0x367a49b, 0x360179b, 0x5c862d6, 0x30dc1af, 0x7797d1, 0x221201f, 0x5eb4345, 0x5e9baad, 0x39b3b47, 0x32f0b8f, 0x48554de, 0x3e8b5e8, 0x5e4f31f, 0x48a53a6, 0x270aba9, 0x240cb06, 0x574030a, 0x1618f3a, 0x271259f, 0x3a306e5, 0x1d33b46, 0x17c29b5, 0x1cf02f4, 0xeb896b ];
console.log($)
var a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r, s, t, u, v, w, x, y, z;
			function check() {
				var answer = "aaa";
				var correct = (function() {
					try {
						h = new MersenneTwister(parseInt(btoa(answer[_[$[6]]](0, 4)), 32)); //取answer前4位，取base64解码后的按32进制转为数字，如果第一位不能转为数字，返回NAN

            			e = h[_[$[""+ +[]]]]()*(""+{})[_[0x4728122]](0xc); for(var _1=0; _1<h.mti; _1++) { e ^= h.mt[_1]; }
            			// e = h['random']()*99; for(var _1=0; _1<1; _1++) { e ^= h.mt[_1]; }   h.mt是根据输入的随机
           				
           				l = new MersenneTwister(e);
						l.random(); l.random(); l.random();
						o = answer.split("_"); //answer按_分割

						i = l.mt[~~(h.random()*35725343)%0xff];
						// i = 941574242； h.random()每次调用都会不同，所以这里i是死数字

						s = ["0x" + i[_[$[$.length/2]]](0x10), "0x" + e[_[$[$.length/2]]](16).split("-")[1]];
						// s = ["0x" + 381f4862, "0x" + e['toString'](0o20).split("-")[1]];  e是和输入有关的随机数

						e =- (this['eval'](_[$[31]](o[1])) ^ s[0]); if (-e != 941564184) return false;

						// _[$[31]] function (_9) { var _8 = []; for (var _a = 0, _b = _9.length; _a < _b; _a++) { _8.push(Number(_9.charCodeAt(_a)).toString(16)); } return "0x" + _8.join(""); }
						// o[1]是下滑线的后半段   e已知 0x381f2118  s[0]已知 0x381f4862   输入为0x697a

						e ^= (this['eval'](_[$[31]](o[2])) ^ s[1]); if (-e != 48879197) return false; e -= 0x352c4a9b;
						console.log("e3:"+e);
            			console.log("e3的参数"+$[22]);
						//e=-48879197(-0x2e9d65d)  e2=-941564184(-0x381f2118)  s[1]=0x3a9b9622   s[1]和输入的异或为0x3af6f74b  输入为0x6d6169

						t = new MersenneTwister(Math.sqrt(-e));
						h.random();
						a = l.random();
						t.random();
						y = [ 0xb3f970, 0x4b9257a, 0x46e990e ].map(function(i) { return $['indexOf'](i)+ +1+ -1- +1; });
						//y：1,2,4

						o[0] = o[0].substring(5); o[3] = o[3].substring(0, o[3].length - 1);
						//o[0]是前五位以后的，也就是sctf{后的，o[3]是从第三个下划线之后到}的
						u = ~~~~~~~~~~~~~~~~(a * i);
						//a和i这里都是固定数字u:31251000
						a = parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(_[$[31]](o[0]));

						//_[$[23]]是函数  function (_c, _d) { var _e = ""; for(var _f=0; _f<_d; _f++) { _e += _c; } return _e; }  第一段和第四段的最长值有几个就返回几个1连起来
						//_[$[31]]       function (_9) { var _8 = []; for (var _a = 0, _b = _9.length; _a < _b; _a++) { _8.push(Number(_9.charCodeAt(_a)).toString(16)); console.log("_8:"+_8);} return "0x" + _8.join(""); }
						//这里o[3]更长，应该是11位，所以是88573，难道12位？265720 13位呢？797161

						r = (h.random() * l.random() * t.random()) / (h.random() * l.random() * t.random());
						e ^= ~r;
						r = (h.random() / l.random() / t.random()) / (h.random() * l.random() * t.random());
						e ^= ~~r;
						//这里e:940974335
						a += _[$[31]](o[3].substring(o[3].length - 2)).split("x")[1];
						//o[3]的最后两位
						d = parseInt(a, 16) == (Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ "");
						//d = parseInt(a, 16) ==  "65531" + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + "2015";
						// parseInt(a,16) 这里是9035121761089634           6553164538462015   0x17481184783f3f  1748202035  
						//  1748078178513f
						//说明o[3]的倒数第三位决定了o[0]，这里倒数第三位首先不能带有字母，其次混入字符串中，转16进制，除后四位以外不能有字母

						i = 0xff;
						n = (f = _[$[23]](o[3].charAt(o[3].length - 4), 3)) == o[3].substring(1, 4);
						// f 是o[3]的倒数第4位重复3遍和o[3]234位相等

						g = 111;
						t = _[$[23]](o[3].charAt(3), 3) == o[3].substring(5, 8) && (o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab;
						//o[3]的第四位重复三遍和o[3]的678位相同，o[3]第2位的阿斯克码-2×o[0]第1位的阿斯克码==0x32ab

						h = ((g ^ e ^ 96) & i).toString(16);
						//e=940974335 h=f0

						i = o[3].split(f).join("");

						s = i.substring(0, 2) == h;

						return (n & t & s) === 1 || (n & t & s) === true;
					} catch (e) {
						console.log("screw you");
						return false;
					}
          console.log(correct);
				})();

       
			};

check();




sctf{wh3r3_iz_mai_fooo0oood??}   那个d算的方式根本忽略了第一位....而且没有d的验证判断（╯－＿－）╯╧╧
```
上面的分析我相信已经很详细了，如果实际做过题目肯定看得懂。
那么最后放上完整的脚本用来辅助验证的，可以直接拖入控制台跑

```

/*
  I've wrapped Makoto Matsumoto and Takuji Nishimura's code in a namespace
  so it's better encapsulated. Now you can have multiple random number generators
  and they won't stomp all over eachother's state.
  
  If you want to use this as a substitute for Math.random(), use the random()
  method like so:
  
  var m = new MersenneTwister();
  var randomNumber = m.random();
  
  You can also call the other genrand_{foo}() methods on the instance.

  If you want to use a specific seed in order to get a repeatable random
  sequence, pass an integer into the constructor:

  var m = new MersenneTwister(123);

  and that will always produce the same random sequence.

  Sean McCullough (banksean@gmail.com)
*/

/* 
   A C-program for MT19937, with initialization improved 2002/1/26.
   Coded by Takuji Nishimura and Makoto Matsumoto.
 
   Before using, initialize the state by using init_genrand(seed)  
   or init_by_array(init_key, key_length).
 
   Copyright (C) 1997 - 2002, Makoto Matsumoto and Takuji Nishimura,
   All rights reserved.                          
 
   Redistribution and use in source and binary forms, with or without
   modification, are permitted provided that the following conditions
   are met:
 
     1. Redistributions of source code must retain the above copyright
        notice, this list of conditions and the following disclaimer.
 
     2. Redistributions in binary form must reproduce the above copyright
        notice, this list of conditions and the following disclaimer in the
        documentation and/or other materials provided with the distribution.
 
     3. The names of its contributors may not be used to endorse or promote 
        products derived from this software without specific prior written 
        permission.
 
   THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
   "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
   LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
   A PARTICULAR PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
   CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
   EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
   PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
   PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
   LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
   NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
   SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 
 
   Any feedback is very welcome.
   http://www.math.sci.hiroshima-u.ac.jp/~m-mat/MT/emt.html
   email: m-mat @ math.sci.hiroshima-u.ac.jp (remove space)
*/

var MersenneTwister = function(seed) {
  if (seed == undefined) {
    seed = new Date().getTime();
  } 
  /* Period parameters */  
  this.N = 624;
  this.M = 397;
  this.MATRIX_A = 0x9908b0df;   /* constant vector a */
  this.UPPER_MASK = 0x80000000; /* most significant w-r bits */
  this.LOWER_MASK = 0x7fffffff; /* least significant r bits */
 
  this.mt = new Array(this.N); /* the array for the state vector */
  this.mti=this.N+1; /* mti==N+1 means mt[N] is not initialized */

  this.init_genrand(seed);
}  
 
/* initializes mt[N] with a seed */
MersenneTwister.prototype.init_genrand = function(s) {
  this.mt[0] = s >>> 0;
  for (this.mti=1; this.mti<this.N; this.mti++) {
      var s = this.mt[this.mti-1] ^ (this.mt[this.mti-1] >>> 30);
   this.mt[this.mti] = (((((s & 0xffff0000) >>> 16) * 1812433253) << 16) + (s & 0x0000ffff) * 1812433253)
  + this.mti;
      /* See Knuth TAOCP Vol2. 3rd Ed. P.106 for multiplier. */
      /* In the previous versions, MSBs of the seed affect   */
      /* only MSBs of the array mt[].                        */
      /* 2002/01/09 modified by Makoto Matsumoto             */
      this.mt[this.mti] >>>= 0;
      /* for >32 bit machines */
  }
}
 
/* initialize by an array with array-length */
/* init_key is the array for initializing keys */
/* key_length is its length */
/* slight change for C++, 2004/2/26 */
MersenneTwister.prototype.init_by_array = function(init_key, key_length) {
  var i, j, k;
  this.init_genrand(19650218);
  i=1; j=0;
  k = (this.N>key_length ? this.N : key_length);
  for (; k; k--) {
    var s = this.mt[i-1] ^ (this.mt[i-1] >>> 30)
    this.mt[i] = (this.mt[i] ^ (((((s & 0xffff0000) >>> 16) * 1664525) << 16) + ((s & 0x0000ffff) * 1664525)))
      + init_key[j] + j; /* non linear */
    this.mt[i] >>>= 0; /* for WORDSIZE > 32 machines */
    i++; j++;
    if (i>=this.N) { this.mt[0] = this.mt[this.N-1]; i=1; }
    if (j>=key_length) j=0;
  }
  for (k=this.N-1; k; k--) {
    var s = this.mt[i-1] ^ (this.mt[i-1] >>> 30);
    this.mt[i] = (this.mt[i] ^ (((((s & 0xffff0000) >>> 16) * 1566083941) << 16) + (s & 0x0000ffff) * 1566083941))
      - i; /* non linear */
    this.mt[i] >>>= 0; /* for WORDSIZE > 32 machines */
    i++;
    if (i>=this.N) { this.mt[0] = this.mt[this.N-1]; i=1; }
  }

  this.mt[0] = 0x80000000; /* MSB is 1; assuring non-zero initial array */ 
}
 
/* generates a random number on [0,0xffffffff]-interval */
MersenneTwister.prototype.genrand_int32 = function() {
  var y;
  var mag01 = new Array(0x0, this.MATRIX_A);
  /* mag01[x] = x * MATRIX_A  for x=0,1 */

  if (this.mti >= this.N) { /* generate N words at one time */
    var kk;

    if (this.mti == this.N+1)   /* if init_genrand() has not been called, */
      this.init_genrand(5489); /* a default initial seed is used */

    for (kk=0;kk<this.N-this.M;kk++) {
      y = (this.mt[kk]&this.UPPER_MASK)|(this.mt[kk+1]&this.LOWER_MASK);
      this.mt[kk] = this.mt[kk+this.M] ^ (y >>> 1) ^ mag01[y & 0x1];
    }
    for (;kk<this.N-1;kk++) {
      y = (this.mt[kk]&this.UPPER_MASK)|(this.mt[kk+1]&this.LOWER_MASK);
      this.mt[kk] = this.mt[kk+(this.M-this.N)] ^ (y >>> 1) ^ mag01[y & 0x1];
    }
    y = (this.mt[this.N-1]&this.UPPER_MASK)|(this.mt[0]&this.LOWER_MASK);
    this.mt[this.N-1] = this.mt[this.M-1] ^ (y >>> 1) ^ mag01[y & 0x1];

    this.mti = 0;
  }

  y = this.mt[this.mti++];

  /* Tempering */
  y ^= (y >>> 11);
  y ^= (y << 7) & 0x9d2c5680;
  y ^= (y << 15) & 0xefc60000;
  y ^= (y >>> 18);

  return y >>> 0;
}
 
/* generates a random number on [0,0x7fffffff]-interval */
MersenneTwister.prototype.genrand_int31 = function() {
  return (this.genrand_int32()>>>1);
}
 
/* generates a random number on [0,1]-real-interval */
MersenneTwister.prototype.genrand_real1 = function() {
  return this.genrand_int32()*(1.0/4294967295.0); 
  /* divided by 2^32-1 */ 
}

/* generates a random number on [0,1)-real-interval */
MersenneTwister.prototype.random = function() {
  return this.genrand_int32()*(1.0/4294967296.0); 
  /* divided by 2^32 */
}
 
/* generates a random number on (0,1)-real-interval */
MersenneTwister.prototype.genrand_real3 = function() {
  return (this.genrand_int32() + 0.5)*(1.0/4294967296.0); 
  /* divided by 2^32 */
}
 
/* generates a random number on [0,1) with 53-bit resolution*/
MersenneTwister.prototype.genrand_res53 = function() { 
  var a=this.genrand_int32()>>>5, b=this.genrand_int32()>>>6; 
  return(a*67108864.0+b)*(1.0/9007199254740992.0); 
} 
/*
CryptoJS v3.1.2
code.google.com/p/crypto-js
(c) 2009-2013 by Jeff Mott. All rights reserved.
code.google.com/p/crypto-js/wiki/License
*/
var CryptoJS=CryptoJS||function(e,m){var p={},j=p.lib={},l=function(){},f=j.Base={extend:function(a){l.prototype=this;var c=new l;a&&c.mixIn(a);c.hasOwnProperty("init")||(c.init=function(){c.$super.init.apply(this,arguments)});c.init.prototype=c;c.$super=this;return c},create:function(){var a=this.extend();a.init.apply(a,arguments);return a},init:function(){},mixIn:function(a){for(var c in a)a.hasOwnProperty(c)&&(this[c]=a[c]);a.hasOwnProperty("toString")&&(this.toString=a.toString)},clone:function(){return this.init.prototype.extend(this)}},
n=j.WordArray=f.extend({init:function(a,c){a=this.words=a||[];this.sigBytes=c!=m?c:4*a.length},toString:function(a){return(a||h).stringify(this)},concat:function(a){var c=this.words,q=a.words,d=this.sigBytes;a=a.sigBytes;this.clamp();if(d%4)for(var b=0;b<a;b++)c[d+b>>>2]|=(q[b>>>2]>>>24-8*(b%4)&255)<<24-8*((d+b)%4);else if(65535<q.length)for(b=0;b<a;b+=4)c[d+b>>>2]=q[b>>>2];else c.push.apply(c,q);this.sigBytes+=a;return this},clamp:function(){var a=this.words,c=this.sigBytes;a[c>>>2]&=4294967295<<
32-8*(c%4);a.length=e.ceil(c/4)},clone:function(){var a=f.clone.call(this);a.words=this.words.slice(0);return a},random:function(a){for(var c=[],b=0;b<a;b+=4)c.push(4294967296*e.random()|0);return new n.init(c,a)}}),b=p.enc={},h=b.Hex={stringify:function(a){var c=a.words;a=a.sigBytes;for(var b=[],d=0;d<a;d++){var f=c[d>>>2]>>>24-8*(d%4)&255;b.push((f>>>4).toString(16));b.push((f&15).toString(16))}return b.join("")},parse:function(a){for(var c=a.length,b=[],d=0;d<c;d+=2)b[d>>>3]|=parseInt(a.substr(d,
2),16)<<24-4*(d%8);return new n.init(b,c/2)}},g=b.Latin1={stringify:function(a){var c=a.words;a=a.sigBytes;for(var b=[],d=0;d<a;d++)b.push(String.fromCharCode(c[d>>>2]>>>24-8*(d%4)&255));return b.join("")},parse:function(a){for(var c=a.length,b=[],d=0;d<c;d++)b[d>>>2]|=(a.charCodeAt(d)&255)<<24-8*(d%4);return new n.init(b,c)}},r=b.Utf8={stringify:function(a){try{return decodeURIComponent(escape(g.stringify(a)))}catch(c){throw Error("Malformed UTF-8 data");}},parse:function(a){return g.parse(unescape(encodeURIComponent(a)))}},
k=j.BufferedBlockAlgorithm=f.extend({reset:function(){this._data=new n.init;this._nDataBytes=0},_append:function(a){"string"==typeof a&&(a=r.parse(a));this._data.concat(a);this._nDataBytes+=a.sigBytes},_process:function(a){var c=this._data,b=c.words,d=c.sigBytes,f=this.blockSize,h=d/(4*f),h=a?e.ceil(h):e.max((h|0)-this._minBufferSize,0);a=h*f;d=e.min(4*a,d);if(a){for(var g=0;g<a;g+=f)this._doProcessBlock(b,g);g=b.splice(0,a);c.sigBytes-=d}return new n.init(g,d)},clone:function(){var a=f.clone.call(this);
a._data=this._data.clone();return a},_minBufferSize:0});j.Hasher=k.extend({cfg:f.extend(),init:function(a){this.cfg=this.cfg.extend(a);this.reset()},reset:function(){k.reset.call(this);this._doReset()},update:function(a){this._append(a);this._process();return this},finalize:function(a){a&&this._append(a);return this._doFinalize()},blockSize:16,_createHelper:function(a){return function(c,b){return(new a.init(b)).finalize(c)}},_createHmacHelper:function(a){return function(b,f){return(new s.HMAC.init(a,
f)).finalize(b)}}});var s=p.algo={};return p}(Math);
(function(){var e=CryptoJS,m=e.lib,p=m.WordArray,j=m.Hasher,l=[],m=e.algo.SHA1=j.extend({_doReset:function(){this._hash=new p.init([1732584193,4023233417,2562383102,271733878,3285377520])},_doProcessBlock:function(f,n){for(var b=this._hash.words,h=b[0],g=b[1],e=b[2],k=b[3],j=b[4],a=0;80>a;a++){if(16>a)l[a]=f[n+a]|0;else{var c=l[a-3]^l[a-8]^l[a-14]^l[a-16];l[a]=c<<1|c>>>31}c=(h<<5|h>>>27)+j+l[a];c=20>a?c+((g&e|~g&k)+1518500249):40>a?c+((g^e^k)+1859775393):60>a?c+((g&e|g&k|e&k)-1894007588):c+((g^e^
k)-899497514);j=k;k=e;e=g<<30|g>>>2;g=h;h=c}b[0]=b[0]+h|0;b[1]=b[1]+g|0;b[2]=b[2]+e|0;b[3]=b[3]+k|0;b[4]=b[4]+j|0},_doFinalize:function(){var f=this._data,e=f.words,b=8*this._nDataBytes,h=8*f.sigBytes;e[h>>>5]|=128<<24-h%32;e[(h+64>>>9<<4)+14]=Math.floor(b/4294967296);e[(h+64>>>9<<4)+15]=b;f.sigBytes=4*e.length;this._process();return this._hash},clone:function(){var e=j.clone.call(this);e._hash=this._hash.clone();return e}});e.SHA1=j._createHelper(m);e.HmacSHA1=j._createHmacHelper(m)})();
/* These real versions are due to Isaku Wada, 2002/01/09 added */
Array.prototype.includes||(Array.prototype.includes=function(a){"use strict";var b=Object(this),c=parseInt(b.length)||0;if(0===c)return!1;var e,d=parseInt(arguments[1])||0;d>=0?e=d:(e=c+d,0>e&&(e=0));for(var f;c>e;){if(f=b[e],a===f||a!==a&&f!==f)return!0;e++}return!1});



var _ = { 0x4c19cff: "random", 0x4728122: "charCodeAt", 0x2138878: "substring", 0x3ca9c7b: "toString", 0x574030a: "eval", 0x270aba9: "indexOf", 0x221201f: function(_9) { var _8 = []; for (var _a = 0, _b = _9.length; _a < _b; _a++) { _8.push(Number(_9.charCodeAt(_a)).toString(16)); console.log("_8:"+_8);} return "0x" + _8.join(""); }, 0x240cb06: function(_2, _3) { var _4 = Math.max(_2.length, _3.length); var _7 = _2 + _3; var _6 = ""; for(var _5=0; _5<_4; _5++) { _6 += _7.charAt((_2.charCodeAt(_5%_2.length) ^ _3.charCodeAt(_5%_3.length)) % _4); } return _6; }, 0x5c623d0: function(_c, _d) { var _e = ""; for(var _f=0; _f<_d; _f++) { _e += _c; } return _e; } };
			
console.log(_)
var $ = [ 0x4c19cff, 0x3cfbd6c, 0xb3f970, 0x4b9257a, 0x1409cc7, 0x46e990e, 0x2138878, 0x1e1049, 0x164a1f9, 0x494c61f, 0x490f545, 0x51ecfcb, 0x4c7911a, 0x29f7b65, 0x4dde0e4, 0x49f889f, 0x5ebd02c, 0x556f342, 0x3f7f3f6, 0x11544aa, 0x53ed47d, 0x381f2118, 0x2e9d65d, 0x5c623d0, 0x32e8f8b, 0x3ca9c7b, 0x367a49b, 0x360179b, 0x5c862d6, 0x30dc1af, 0x7797d1, 0x221201f, 0x5eb4345, 0x5e9baad, 0x39b3b47, 0x32f0b8f, 0x48554de, 0x3e8b5e8, 0x5e4f31f, 0x48a53a6, 0x270aba9, 0x240cb06, 0x574030a, 0x1618f3a, 0x271259f, 0x3a306e5, 0x1d33b46, 0x17c29b5, 0x1cf02f4, 0xeb896b ];
console.log($)
var a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r, s, t, u, v, w, x, y, z;
			function check() {
        var answer = "sctf{wh3r3_iz_mai_fooo0oood??}";
				var correct = (function() {
					try {
						h = new MersenneTwister(parseInt(btoa(answer['substring'](0, 4)), 32));
            e = h['random']()*(""+{})['charCodeAt'](0xc); for(var _1=0; _1<h.mti; _1++) { e ^= h.mt[_1]; }
            console.log("e:"+e);
            l = new MersenneTwister(e);
						l.random(); l.random(); l.random();
            o = answer.split("_");
						i = l.mt[~~(h.random()*35725343)%0xff];
            console.log("i:"+i);
            s = ["0x" + i[_[$[$.length/2]]](0x10), "0x" + e[_[$[$.length/2]]](0o20).split("-")[1]];
            console.log("s:"+s);
            e =- (this[_[$[42]]](_[$[31]](o[1])) ^ s[0]); 
            console.log("e2:"+e);
            if (-e != $[21]) return false;
            
            e ^= (this[_[$[42]]](_[$[31]](o[2])) ^ s[1]); 
            console.log("e3:"+e);
            
            if (-e != $[22]) return false; e -= 0x352c4a9b;
						
            console.log("e4:"+e);
            t = new MersenneTwister(Math.sqrt(-e));
           
						h.random();
						a = l.random();
						t.random();
						y = [ 0xb3f970, 0x4b9257a, 0x46e990e ].map(function(i) { return $[_[$[40]]](i)+ +1+ -1- +1; });
            o[0] = o[0].substring(5); o[3] = o[3].substring(0, o[3].length - 1);
						u = ~~~~~~~~~~~~~~~~(a * i);
						a = parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3) ^ eval(_[$[31]](o[0]));
            console.log("a:"+a);
            console.log("aaaa:"+parseInt(_[$[23]]("1", Math.max(o[0].length, o[3].length)), 3))
            r = (h.random() * l.random() * t.random()) / (h.random() * l.random() * t.random());
						e ^= ~r;
						r = (h.random() / l.random() / t.random()) / (h.random() * l.random() * t.random());
						e ^= ~~r;
            console.log("e5:"+e);
            
						a += _[$[31]](o[3].substring(o[3].length - 2)).split("x")[1];
            console.log("a2:"+a);
            console.log("parseInt(a, 16)要等于的:"+(Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ ""));
            d = parseInt(a, 16) == (Math.pow(2, 16)+ -5+ "") + o[3].charCodeAt(o[3].length - 3).toString(16) + "53846" + (new Date().getFullYear()- +1+ "");
            console.log("d:"+d);
            
            i = 0xff;
						n = (f = _[$[23]](o[3].charAt(o[3].length - 4), 3)) == o[3].substring(1, 4);
						g = 111;
						t = _[$[23]](o[3].charAt(3), 3) == o[3].substring(5, 8) && (o[3].charCodeAt(1)-2) * o[0].charCodeAt(0) == 0x32ab;
        
            
            h = ((g ^ e ^ 96) & i).toString(16);
            console.log("h:"+h);
            console.log("f:"+f);
            
            
            
						i = o[3].split(f).join("");
            console.log("i:"+i);
            console.log("o[3].substring(1, 4)"+o[3].substring(1, 4));
            console.log("o[3].substring(5, 8):"+o[3].substring(5, 8));
            console.log("i.substring(0, 2):"+i.substring(0, 2));
            console.log("o[3]:"+o[3]);
            
						s = i.substring(0, 2) == h;
            
            console.log("s:"+s);
            console.log("t:"+t);
            console.log("n:"+n);
            
						return (n & t & s) === 1 || (n & t & s) === true;
					} catch (e) {
						console.log("screw you");
						return false;
					}
        
				})();
        console.log("correct:"+correct);
       
			};

check();
```