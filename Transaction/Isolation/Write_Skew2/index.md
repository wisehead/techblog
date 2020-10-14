---
title: Clojure的并发（二）Write Skew分析 - 庄周梦蝶 - BlogJava
category: default
tags: 
  - www.blogjava.net
created_at: 2020-09-12 21:04:29
original_url: http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/17/326362.html
---


# Clojure的并发（二）Write Skew分析 - 庄周梦蝶 - BlogJava

[Clojure 的并发（一） Ref和STM](http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/archive/2010/07/14/326027.html)  
[Clojure 的并发（二）Write Skew分析](http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/archive/2010/07/17/326362.html)  
[Clojure 的并发（三）Atom、缓存和性能](http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/archive/2010/07/17/326389.html)  
[Clojure 的并发（四）Agent深入分析和Actor](http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/archive/2010/07/19/326540.html)  
[Clojure 的并发（五）binding和let](http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/archive/2010/07/23/326976.html)  
[Clojure的并发（六）Agent可以改进的地方](http://www.blogjava.net/killme2008/archive/2010/07/30/327606.html "Clojure的并发（六）Agent可以改进的地方")  
[Clojure的并发（七）pmap、pvalues和pcalls](http://www.blogjava.net/killme2008/archive/2010/08/04/327985.html "Clojure的并发（七）pmap、pvalues和pcalls")  
[Clojure的并发（八）future、promise和线程](http://www.blogjava.net/killme2008/archive/2010/08/08/328230.html "Clojure的并发（八）future、promise和线程")  
  
     在[介绍Ref的上一篇blog](http://www.blogjava.net/killme2008/archive/2010/07/14/326027.html)提到，基于snapshot做隔离的MVCC实现来说，有个现象，叫[写偏序——Write Skew](http://en.wikipedia.org/wiki/Snapshot_isolation)。根本的原因是由于每个事务在更新过程中无法看到其他事务的更改的结果，导致各个事务提交之后的最终结果违反了一致性。为了理解这个现象，最好的办法是在代码中复现这个现象。考虑下列这个场景：  
   屁民Peter有两个账户account1和account2，简称为A1和A2，这两个账户各有100块钱，一个显然的约束就是这两个账户的余额之和必须大于或者等于零，银行肯定不能让你赚了去，你也怕成为下个许霆。现在，假设有两个事务T1和T2，T1从A1提取200块钱，T2则从A2提取200块钱。如果这两个事务按照先后顺序进行，后面执行的事务判断A1+A2-200>=0约束的时候发现失败，那么就不会执行，保证了一致性和隔离性。但是基于多版本并发控制的Clojure，这两个事务完全可能并发地执行，因为他们都是基于一个当前账户的快照做更新的， 并且在更新过程中无法看到对方的修改结果，T1执行的时候判断A1+A2-200>=0约束成立，从A1扣除了200块；同样，T2查看当前快照也满足约束A1+A2-200>=0，从A2扣除了200块，问题来了，最终的结果是A1和A2都成-100块了，身为屁民的你竟然从银行多拿了200块，你等着无期吧。  
  
   现在，我们就来模拟这个现象，定义两个账户：  
  

;;两个账户，约束是两个账户的余额之和必须\>=0  
(def account1 (ref 100))  
(def account2 (ref 100))

  
   定义一个取钱方法：  

;;定义扣除函数  
(defn deduct \[account n other\]  
      (dosync   
          (if (\>= (+ (\- @account n) @other) 0)  
              (alter account \- n))))

  
   其中account是将要扣钱的帐号，other是peter的另一个帐号，在执行扣除前要满足约束@account-n+@other>=0  
  
   接下来就是搞测试了，各启动N个线程尝试从A1和A2扣钱，为了尽快模拟出问题，使得并发程度高一些，我们将线程设置大一些，并且使用java.util.concurrent.CyclicBarrier做关卡，测试代码如下：  
  

;;设定关卡  
(def barrier (java.util.concurrent.CyclicBarrier. 6001))  
;;各启动3000个线程尝试去从账户1和账户2扣除200  
(dotimes \[\_ 3000\] (.start (Thread. #(do (.await  barrier) (deduct account1 200 account2) (.await  barrier)))))  
(dotimes \[\_ 3000\] (.start (Thread. #(do (.await  barrier) (deduct account2 200 account1) (.await  barrier)))))  
  
(.await barrier)  
  
(.await barrier)  
;;打印最终结果  
(println @account1)  
(println @account2)

  
     线程里干了三件事情：首先调用barrier.await尝试突破关卡，所有线程启动后冲破关卡，进入扣钱环节deduct，最后再调用barrier.await用于等待所有线程结束。在所有线程结束后，打印当前账户的余额。  
  
     这段代码在我的机器上每执行10次左右都至少有一次打印：  

\-100  
\-100

     
    这表示A1和A2的账户都欠下了100块钱，完全违反了约束条件，法庭的传票在召唤peter。  
  
    那么怎么防止write skew现象呢？如果我们能在事务过程中保护某些Ref不被其他事务修改，那么就可以保证当前的snapshot的一致性，最终保证结果的一致性。通过ensure函数即可保护Ref，稍微修改下deduct函数：  

(defn deduct \[account n other\]  
      (dosync (ensure account) (ensure other)  
          (if (\>= (+ (\- @account n) @other) 0)  
              (alter account \- n))))

  
   在执行事务更新前，先通过ensure保护下account和other账户不被其他事务修改。你可以再多次运行看看，会不会再次打印非法结果。  
  
   [上篇blog](http://www.blogjava.net/killme2008/archive/2010/07/14/326027.html)最后也提到了一个士兵巡逻的例子来介绍write skew，我也写了段代码来模拟那个例子，有兴趣可以跑跑，非法结果是三个军营的士兵之和小于100(两个军营最后只剩下25个人）。  
  

;1号军营  
(def g1 (ref 45))  
;2号军营  
(def g2 (ref 45))  
;3号军营  
(def g3 (ref 45))  
;从1号军营抽调士兵  
(defn dispatch\-patrol\-g1 \[n\]  
    (dosync   
      (if (\> (+ (\- @g1 n) @g2 @g3) 100)  
          (alter g1 \- 20)  
        ))  
      )  
;从2号军营抽调士兵  
(defn dispatch\-patrol\-g2 \[n\]  
    (dosync   
      (if (\> (+ @g1 (\- @g2 n) @g3) 100)  
          (alter g2 \- 20)  
        ))  
      )  
;;设定关卡  
(def barrier (java.util.concurrent.CyclicBarrier. 4001))  
;;各启动2000个线程尝试去从1号和2号军营抽调20个士兵  
(dotimes \[\_ 2000\] (.start (Thread. #(do (.await  barrier) (dispatch-patrol-g1 20) (.await  barrier)))))  
(dotimes \[\_ 2000\] (.start (Thread. #(do (.await  barrier) (dispatch-patrol-g2 20) (.await  barrier)))))  
;(dotimes \[\_ 10\] (.start (Thread. #(do (.await  barrier) (dispatch-patrol-g3 20) (.await  barrier)))))  
  
(.await barrier)  
  
(.await barrier)  
;;打印最终结果  
(println @g1)  
(println @g2)  
(println @g3)

---------------------------------------------------


原网址: [访问](http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/17/326362.html)

创建于: 2020-09-12 21:04:29

目录: default

标签: `www.blogjava.net`

