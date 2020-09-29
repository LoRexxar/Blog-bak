---
title: 聊聊hctf2016
date: 2017-01-26 00:00:42
tags:
- Blogs
- eassy
- hctf
categories:
- Blogs
---


hctf2016在11月底落幕了，零零散散已经过去了2个月了，时至今日，才算是有空收拾下一下去年的得失...

过年了总想写点儿什么，下面的东西就随便说说，祝大家新年大吉x.
<!--more-->

# 聊点儿有的没得 #

相信很多人都知道当年十一期间闹得沸沸扬扬的L-CTF和XDCTF的事，很多事情的发生让人措手不及，所以西电的大佬们选择了这样的方式来解决问题，而同样的事情其实当时正在发生在杭电HDUISA的身上...但就如当年在知乎上发的一样，为了能让杭电的CTF走的更远，我们选择了今天这条路。

早在16年的5、6月份时候，杭电成立了浙江第二个网络空间安全学院，但没想到的是，在整个信息安全专业被剥离到网安学院的同时，信息安全协会（HDUISA）却留在了通信学院，并在网安学院领导的安排下，成立了新的网络空间安全协会...在很多事情都不知情的情况下，安协和网协就被摆在了对立面上，两个协会，一个趁着学院新成立全校的支持开始大范围招新，我们虽有多年底蕴却又无可奈何，被迫扩大了招新范围...

幸运的是，不管网安学院带着多么大的恶意对待我们，作为两个协会的负责人，我们不希望把精力花费在无意义的内斗上，在各种取舍之后，HDUISA这个名字成了历史，Vidar-Team带着希望出现...

[http://vidar.club/](http://vidar.club/)

![{S8TNFGAAQX~E`L9I[NXX1G.png-867.4kB][1]

这时候应该正是十一的时候...hitcon应该也是Vidar正式成立的第一场比赛，也就是这个时候开始，我们承诺HCTF一定会举办下去...

只是没想到，HCTF确实成功办下来了，但是困难重重，最后还被迫取消了线下赛。

不知道有多少人去办比赛有概念，要想办一个优秀的比赛（这个比我装了我不管），比如我们是11月底比赛，从6月学期结束后就要开始收集思路，构想比赛赛制（这个可能是HCTF特有的），然后暑假期间就要沟通赞助，学期开始也就是9月就要开始出题，并且写比赛平台（平台的开发周期很长，还面临一些特殊情况），比赛开始前一个月左右，赛制确定，题目确定，平台确定，比赛开始前一周，所有题目就位定分...提前一周左右开始报名宣传...提前三天左右，开始测试...比赛开始要一直盯着题目，尤其是平台和web题目，要全程看着，几乎不能休息。

不知道有多少人看明白了其中的关键，由于每年暑假都是一次换届，学弟还不够强，学长要毕业实习，所以这个时候人手会非常缺，所以下半年的高校比赛几乎只有我们和西电（别和我提什么WD、xxxxxxxxx），去年有个ZCTF今年也推迟了。

时间回到十月，由于网安学院的事情让我们措手不及，所以等到招新结束，Vidar成立，很多事情才开始变得稳定了起来，我们终于腾出手来去准备HCTF了，首先我们就遇到了第一个问题，赞助问题！

熟悉HDUISA的人一定知道我们比赛之前一直叫做“安恒杯”，但从去年开始，安恒作为我们的赞助方，试图通过我们的比赛宣传旗下的各个产品，非常反感的我们找了新的赞助方，一个是最后合作的赛宁（XCTF），还有一个和我们很熟悉的公司。出于各方面的原因，我们最后选择了和赛宁合作...

这个时候没想到出现了一些插曲，问鼎杯在找我们要了一批题目之后，在宣传比赛的时候冠上了第七届HCTF的名头...所幸的是，虽然办问鼎杯的领导不讲理，但是网安学院的院长很讲理，所以同意去掉HCTF，比赛能够继续筹备下去。

我们都以为事情总算能顺利的进行下去了，可没想到的是赛宁不但提出要使用他们的比赛平台（就是你们比赛时见到的那个），还大幅缩减了策划中的预算（这成了线下赛没能举办的导火索），以至于在比赛开始前一周，我们仍在争论使用赛宁平台的不可预知问题（当时确定了HCTF2016的下层制度、反作弊制度、动态分数制度），最后结局是我们出人帮助他们解决问题。

而我们开发2月的CTF平台被迫雪藏，只有报名页可以看到冰山一角

![05VA]58W1VFLNSTO67ZL2Z0.png-123.9kB][2]

这其实是个非常棒的平台，为了不留遗憾，这个比赛平台会进行开发后开源

也许你以为事情就这样结束了，但事实上并没有，因为虽然困难，但这时候我们仍希望能举办线下赛，谁都没想到的是，通信学院的领导不干了，申请过校内比赛的人应该知道，这其实是一个很长的申请流程，但因为各种原因，比赛前不到一周，我们才确定了合作方式...于是通信学院的领导拒绝了我们申请比赛的希望，也就意味着，我们不能使用校内的任何东西...

为了能保证线上赛的顺利，我们找了一个学长的公司，也就是比赛页面能看到的萌新科技

![QQ图片20161125163802.png-36.7kB][3]

用学长的公司担任法人代表签了赞助合同，把整个hctf独立于学校外，并和赞助商重新沟通了赞助，修改了奖品，至此，hctf筹备总算是结束了，虽然经历了很多也遇到了很多困难，但结局总是好的。

![14806084705580.png-363.8kB][4]

所幸的是，hctf还是那个hctf，从8年前开始举办第一届hctf开始，hctf从校内走向全国，从一个普通的校内自娱自乐的比赛，到几年前享誉全国的hctf，hctf总算是没让大部分人失望，我们坚持，ctf虽然是一个竞赛，但仍是一个hack game，比起最后的成绩来说，学习新的知识和体会hack的乐趣才是ctf真正的意义...

不知道这样的ctf还让大家满意吗？将来的hctf会接着和世界接轨，明年的hctf应该会登上ctftime，继续发光发热...

# 聊一些比赛背后的数据 #

关于比赛规则，从去年的金币系统开始，我们开始发现，除了传统的ctf解题模式以外，有很多方式让比赛有一些不一样的体验，但金币系统仍然存在很多缺陷，由于没有题目类型，所以可能买错了题目会导致比赛进程大幅度脱节，即使我们推出了打折和金币赠送的方式，仍然不能很好的避免这样的问题。

所以今年我们构想了下层制度，原本的下层制度应该有6层，还配有下层卡，以防止因为题目过于脑洞或者别的导致队伍卡死进度。

经过反复的斟酌，才有了今天的下层制度...5层区分，并把3、4层过关条件改成全部题目的部分...

至于反作弊系统和今年从defcon借鉴的动态分数制度...就没什么可解释的了...最后放一些数据。

成功登陆过的队伍有412支，完成杂项签到的队伍数为235。

完成web签到题目的队伍数为226，结束时还在第一层的队伍有184，在第二层的有168，在第三层的有30，在第四层的有15支，能进入第五层的只有15支。

唯二没有人解出来的题目是Crypto So Amazing和5-days。

最出乎意料的是，浙大AAA在开赛7小时第一个进入了第四层，FlappyPig在14小时第二个进入第四层，而NU1L在半小时后第三个进入第四层。

队伍FlappyPig在开赛24小时第一个进入了第五层！ 在接下来的短时间内陆续有七八只队伍进入了第五层...   

# 最后聊一聊encore time #

在比赛结束之前，一方面为了能让比赛更激烈，一方面也出于想要收集一些信息的想法，于是提前两小时放出了encore time，最后有138位提交了表单，虽然不能代表所有的参赛者，但起码也是在比赛结束时仍在关注我们的参赛者。

![image_1b7b41qtq15ht1870jd030lbm131.png-40.7kB][5]

大部分人还是喜欢我们的比赛的。

![image_1b7b4350i5b11kir1q5a16n519c33e.png-34.3kB][6]

其实不是太有代表性，因为提交表单的人大部分都是最后时间没什么事做的...

![image_1b7b44g9kp3irmg122psg4f6s3r.png-31.8kB][7]

意外的是，动态积分制度还有30%的人不喜欢...

![image_1b7b45ntpn04inord7198923748.png-41.6kB][8]

还有30%的人喜欢，还可以

![image_1b7b46isr80j1ume8fgud62e04l.png-33.1kB][9]

赛前我们一直争论的平台问题还是影响了参赛者

![image_1b7b47ovn1r1k1m1c1f711kin7ip52.png-37.3kB][10]

难得的是，我们倾尽心血的反作弊制度得到了一致的好评

![image_1b7b492psr8s1jfn92hr921flc5f.png-38kB][11]

这问题好像没什么意义...

![image_1b7b4a7p11bk11g791uar1vu0k945s.png-48.9kB][12]

对题目差评的人不太多

碎碎叨叨的也写了一天，就这样吧o(*^▽^*)┛


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/07r1hybqwo1eastp1z6qy1r6/%7BS8TNFGAAQX~E%60L9I%5BNXX1G.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/4fhrafb07xgsbidsk9acyn74/05VA%5D58W1VFLNSTO67ZL2Z0.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/6c2ehzgp96udrq8niusdi4wo/QQ%E5%9B%BE%E7%89%8720161125163802.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/gktrckaybu2ip2cqh18a7z8w/14806084705580.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/m25gnm435fz9y182rttysojv/image_1b7b41qtq15ht1870jd030lbm131.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/xw5j26u0ijsbaadm3u0zf96s/image_1b7b4350i5b11kir1q5a16n519c33e.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/eq5n1qcksvvj4yl4evroidw2/image_1b7b44g9kp3irmg122psg4f6s3r.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/xgy8b75gy1bkdlwurm6r2z40/image_1b7b45ntpn04inord7198923748.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/ednz1fi549q5unaua7unkcjt/image_1b7b46isr80j1ume8fgud62e04l.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/85bms290tpvoowchhofv3jqv/image_1b7b47ovn1r1k1m1c1f711kin7ip52.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/9id24a2lky52s5gl22e5w2zd/image_1b7b492psr8s1jfn92hr921flc5f.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/l4627dqtuxmj14nhh9ru3rh3/image_1b7b4a7p11bk11g791uar1vu0k945s.png