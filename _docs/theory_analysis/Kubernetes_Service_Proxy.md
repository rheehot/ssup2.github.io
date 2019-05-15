---
title: Kubernetes Service Proxy
category: Theory, Analysis
date: 2019-05-06T12:00:00Z
lastmod: 2019-05-06T12:00:00Z
comment: true
adsense: true
---

Kubernetes는 iptables, IPVS, Userspace 3가지 Mode의 Service Proxy를 지원하고 있다. 각 Mode에 따른 Service Network를 분석한다.

### 1. iptables Mode

![[그림 1] iptables Mode에서 Service Packet 경로]({{site.baseurl}}/images/theory_analysis/Kubernetes_Service_Proxy/iptables_Mode_Service_Packet_Path.PNG){: width="700px"}

{% highlight text %}
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !192.167.0.0/16       10.103.1.234         /* default/my-nginx-cluster: cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-52FY5WPFTOHXARFK  tcp  --  *      *       0.0.0.0/0            10.103.1.234         /* default/my-nginx-cluster: cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !192.167.0.0/16       10.97.229.148        /* default/my-nginx-nodeport: cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-6JXEEPSEELXY3JZG  tcp  --  *      *       0.0.0.0/0            10.97.229.148        /* default/my-nginx-nodeport: cluster IP */ tcp dpt:80
    0     0 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 1] iptables Mode의 KUBE-SERVICE </figcaption>
</figure>

{% highlight text %}
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx-nodeport: */ tcp dpt:30915
    0     0 KUBE-SVC-6JXEEPSEELXY3JZG  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx-nodeport: */ tcp dpt:30915 
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 2] iptables Mode의 KUBE-NODEPORTS </figcaption>
</figure>

{% highlight text %}
Chain KUBE-SVC-6JXEEPSEELXY3JZG (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-6HM47TA5RTJFOZFJ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.33332999982
    0     0 KUBE-SEP-AHRDCNDYGFSFVA64  all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-BK523K4AX5Y34OZL  all  --  *      *       0.0.0.0/0            0.0.0.0/0      
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 3] iptables Mode의 KUBE-SVC-XXX </figcaption>
</figure>

{% highlight text %}
Chain KUBE-SEP-QQATNRPNVZFKMY6D (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       192.167.1.138        0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:192.167.1.138:53 
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 4] iptables Mode의 KUBE-SEP-XXX </figcaption>
</figure>

{% highlight text %}
Chain KUBE-MARK-MASQ (23 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000 
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 5] iptables Mode의 KUBE-MARK-MASQ </figcaption>
</figure>

{% highlight text %}
Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match
0x4000/0x4000 
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 6] iptables Mode의 KUBE-POSTROUTING </figcaption>
</figure>

iptables Proxy Mode는 Kubernetes가 이용하는 Default Proxy Mode이다. [그림 1]은 iptables Mode에서 Service로 전송되는 Packet의 경로를 나타내고 있다. [NAT Table 1] ~ [NAT Table 6]는 [그림 1]의 각 NAT Table의 실제 내용을 보여주고 있다. [그림 1]의 NAT Table들은 Kubernetes Cluster를 구성하는 모든 Node에 동일하게 설정된다. 따라서 Kubernetes Cluster를 구성하는 어느 Node에서도 Service로 Packet을 전송할 수 있다.

대부분의 Pod에서 전송된 Packet은 Pod의 veth를 통해서 Host의 Network Namespace로 전달되기 때문에 Packet은 PREROUTING Table에 의해서 KUBE-SERVICE Table로 전달된다. Host의 Network Namespace를 이용하는 Pod 또는 Host Process에서 전송한 Packet은 OUTPUT Table에 의해서 KUBE-SERVICE Table로 전달된다. KUBE-SERVICE Table에서 Packet의 Dest IP와 Dest Port가 ClusterIP Service의 IP와 Port와 일치한다면, 해당 Packet은 일치하는 ClusterIP Service의 NAT Table인 KUBE-SVC-XXX Table로 전달된다. Packet의 Dest IP가 Localhost인 경우에는 해당 Packet은 KUBE-NODEPORTS Table로 전달된다.

KUBE-NODEPORTS Table에서 Packet의 Dest Port가 NodePort Service의 Port와 일치하는 경우 해당 Packet은 NodePort Service의 NAT Table인 KUBE-SVC-XXX Table로 전달된다. KUBE-SVC-XXX Table에서는 iptables의 statistic 기능을 이용하여 Packet은 Service를 구성하는 Pod들로 랜덤하고 균등하게 Load Balancing하는 역활을 수행한다. [NAT Table 3]에서 Service는 3개의 Pod으로 구성되어 있기 때문에 3개의 KUBE-SEP-XXX Table로 Packet이 랜덤하고 균등하게 Load Balancing 되도록 설정되어 있는것을 확인할 수 있다. KUBE-SEP-XXX Table에서 Packet은 Container IP 및 Service에서 설정한 Port로 DNAT를 수행한다. Container IP로 **DNAT**를 수행한 Packet은 CNI Plugin을 통해 구축된 Container Netwokr를 통해서 해당 Container에게 전달된다.

Service로 전달되는 Packet은 iptables의 DNAT를 통해서 Pod에게 전달되기 때문에, Pod에서 전송한 응답 Packet의 Src IP는 Pod의 IP가 아닌 Service의 IP로 **SNAT**되어야 한다. iptables에는 Serivce를 위한 SNAT Rule이 명시되어 있지 않다. 하지만 iptables는 Linux Kernel의 **Conntrack** (Connection Tracking)의 TCP Connection 정보를 바탕으로 Service Pod으로부터 전달받은 Packet을 SNAT한다.

#### 1.1. Hairpinning

![[그림 2] iptables Mode에서 Hairpinning이 적용된 Packet 경로]({{site.baseurl}}/images/theory_analysis/Kubernetes_Service_Proxy/iptables_Mode_Hairpinning.PNG){: width="650px"}

KUBE-MARK-MASQ Table은 Packet Masquerade를 위해서 Packet에 Marking을 수행하는 Table이다. Marking된 Packet은 KUBE-POSTROUTING Table에서 Masquerade 된다. 즉 Packet의 Src IP가 Host의 IP로 SNAT 된다. Masquerade가 필요한 경우중 하나는 Pod에서 자신이 소속되어있는 Service의 IP로 Packet을 전송하여 자기 자신에게 Packet이 돌아올 경우이다. [그림 2]의 왼쪽은 이러한 경우를 나타내고 있다. Packet은 DNAT되어 Packet의 Src IP와 Dest IP는 모두 Pod의 IP가 된다. 따라서 Pod에서 돌아온 Packet에 대한 응답을 보낼경우, Packet은 Host의 NAT Table을 거치지 않고 Pod안에서 처리되기 때문에 SNAT가 수행되지 않는다.

Masquerade를 이용하면 Pod에게 돌아온 Packet을 강제로 Host에게 넘겨 SNAT가 수행되도록 만들 수 있다. 이처럼 Packet을 고의적으로 우회하여 받는 기법을 Hairpinning이라고 부른다. [그림 2]의 오른쪽은 Masqurade를 이용하여 Hairpinning을 적용한 경우를 나타내고 있다. KUBE-SEP-XXX Table에서 Packet의 Src IP가 DNAT 하려는 IP와 동일한 경우, 즉 Pod이 Service로 전송한 Packet을 자기 자신이 받을경우 해당 Packet은 KUBE-MARK-MASQ Table을 거쳐 Marking이 되고 KUBE-POSTROUTING Table에서 Masquerade 된다. Pod이 받은 Packet의 Src IP는 Host의 IP로 설정되어 있기 때문에 Pod의 응답은 Host의 NAT Table 전달되고, 다시 SNAT, DNAT 되어 Pod에게 전달된다. 

### 2. Userspace Mode

![[그림 3] Userspace Mode에서 Service Packet 경로]({{site.baseurl}}/images/theory_analysis/Kubernetes_Service_Proxy/Userspace_Mode_Service_Packet_Path.PNG){: width="700px"}

{% highlight text %}
Chain KUBE-PORTALS-CONTAINER (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            10.103.1.234         /* default/my-nginx-cluster: */ tcp dpt:80 redir ports 46635
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            10.100.15.169        /* default/my-nginx-nodeport: */ tcp dpt:80 redir ports 32847
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 7] Userspace Mode의 KUBE-PORTALS-CONTAINER</figcaption>
</figure>

{% highlight text %}
Chain KUBE-NODEPORT-CONTAINER (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx-nodeport: */ tcp dpt:30915 redir ports 32847 
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 8] Userspace Mode의 KUBE-NODEPORT-CONTAINER</figcaption>
</figure>

{% highlight text %}
Chain KUBE-PORTALS-HOST (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            10.103.1.234         /* default/my-nginx-cluster: */ tcp dpt:80 to:172.35.0.100:46635
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            10.97.229.148        /* default/my-nginx-nodeport: */ tcp dpt:80 to:172.35.0.100:32847
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 9] Userspace Mode의 KUBE-PORTALS-HOST</figcaption>
</figure>

{% highlight text %}
Chain KUBE-NODEPORT-HOST (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx-nodeport: */ tcp dpt:30915 to:172.35.0.100:32847
{% endhighlight %}
<figure>
<figcaption class="caption">[NAT Table 10] Userspace Mode의 KUBE-NODEPORT-HOST</figcaption>
</figure>

Userspace Proxy Mode는 Kubernetes가 처음으로 제공했던 Proxy Mode이다. Userspace에서 동작하는 kube-proxy가 Service Proxy 역활을 수행한다. 현재는 iptables Mode에 비해서 떨어지는 성능 때문에 잘 이용되지 않고 있다. [그림 3]은 Userspace Mode에서 Service로 전송되는 Packet의 경로를 나타내고 있다. [NAT Table 7] ~ [NAT Table 10]은 [그림 3]의 각 NAT Table의 실제 내용을 보여주고 있다. [그림 3]의 NAT Table과 kube-proxy는 Kubernetes Cluster를 구성하는 모든 Node에 동일하게 설정된다. 따라서 Kubernetes Cluster를 구성하는 어느 Node에서도 Service로 Packet을 전송할 수 있다.

대부분의 Pod에서 전송된 Packet은 Pod의 veth를 통해서 Host의 Network Namespace로 전달되기 때문에 Packet은 PREROUTING Table에 의해서 KUBE-PORTALS-CONTAINER Table로 전달된다. KUBE-PORTALS-CONTAINER Table에서 Packet의 Dest IP와 Dest Port가 ClusterIP Service의 IP와 Port와 일치한다면, 해당 Packet은 kube-proxy로 **Redirect**된다. Packet의 Dest IP가 Localhost인 경우에는 해당 Packet은 KUBE-NODEPORT-CONTAINER Table로 전달된다. KUBE-NODEPORT-CONTAINER Table에서 Packet의 Dest Port가 NodePort Service의 Port와 일치하는 경우 해당 Packet은 kube-proxy로 **Redirect**된다.

Host의 Network Namespace를 이용하는 Pod 또는 Host Process에서 전송한 Packet은 OUTPUT Table에 의해서 KUBE-PORTALS-HOST Table로 전달된다. 이후의 KUBE-PORTALS-HOST, KUBE-NODEPORT-HOST Table에서의 Packet 처리과정은 KUBE-PORTALS-CONTAINER, KUBE-NODEPORT-CONTAINER Table에서의 Packet 처리와 유사하다. 차이점은 Packet을 Packet을 Redirect하지 않고 **DNAT**를 수행한다는 점이다.

**Redirect, DNAT를 통해서 Service로 전송한 모든 Packet은 kube-proxy로 전달된다.** kube-proxy가 받는 Packet의 Dest Port 하나당 하나의 Service가 Mapping 되어있다. 따라서 kube-proxy는 Redirect, NAT된 Packet의 Dest Port를 통해서 해당 Packet이 어느 Service로 전달되어야 하는지 파악할 수 있다. kube-proxy는 전달받은 Packet을 Packet이 전송되어야하는 Service에 속한 다수의 Pod에게 균등하게 Load Balancing 하여 다시 전송한다. kube-proxy는 Host의 Network Namespace에서 동작하기 때문에 kube-proxy가 전송한 Packet 또한 Service NAT Table들을 거친다. 하지만 kube-proxy가 전송한 Packet의 Dest IP는 Pod의 IP이기 때문에 해당 Packet은 Service NAT Table에 의해서 변경되지 않고 Pod에게 직접 전달된다.

### 3. IPVS Mode

### 4. 참조

* [https://www.slideshare.net/Docker/deep-dive-in-container-service-discovery](https://www.slideshare.net/Docker/deep-dive-in-container-service-discovery)
* [http://www.system-rescue-cd.org/networking/Load-balancing-using-iptables-with-connmark/](http://www.system-rescue-cd.org/networking/Load-balancing-using-iptables-with-connmark/)
* [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)