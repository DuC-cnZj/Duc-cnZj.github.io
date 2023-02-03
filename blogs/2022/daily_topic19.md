---
title: k8s 每日一问 - nginx ingress local traffic = Local 问题总结以及 externalTrafficPolicy=Local 原理
date: '2022-12-09 17:12:42'
sidebar: true
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

## nginx ingress local traffic = Local 问题总结

1. 从外部通过域名访问app服务: 这样是ok的,因为外部进来总是通过 lb 转发到服务， lb 的 backend 总是对应到有 nginx pod 的节点
2. 从内部pod通过**域名**访问app服务
   - pod 和 nginx 在同一个节点上: <font color=green>ok</font>
   - pod 和 nginx 不在同一个节点上: <font color=red>失败</font>, 因为pod先去解析域名，得到nginx外部ip地址，又因为外部流量策略是Local，使得 iptables 规则访问的域名ip只会转发到和pod所在节点上的nginx 地址， 所以此时访问 nginx 必须是 pod 所在 node 上的 nginx，如果没有就会丢弃流量
             1. podA 域名访问 app
             2.  得到域名的外部 ip，比如 1.2.3.4
             3. 因为 nginx svc 流量策略是 Local, 所以 1.2.3.4 在podA所在节点node1的路由规则，只会把流量转发到节点node1上的 nginx pod, 因为node1没有nginx pod，所以该部分流量就会被丢弃


## externalTrafficPolicy=Local 原理

> [-m addrtype --src-type LOCAL 解释](https://blog.csdn.net/qq_20817327/article/details/113770498)
>
> 详见 kube-proxy 源码 iptabels/proxier_test.go 代码
>
> 当为 Local 时，iptables nat 只会使用 KUBE-SVL-XXXX 的规则，这个规则只包含了 当前节点上的 pod ip`-A KUBE-SVL-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 -> 10.180.2.1:80" -j KUBE-SEP-ZX7GRIZKSNUQ3LAJ`
>
> 如果当前节点上没有 svc 对应的 pod 则会显示`-A KUBE-SVL-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 has no local endpoints" -j KUBE-MARK-DROP`

```go
func TestOnlyLocalExternalIPs(t *testing.T) {
	ipt := iptablestest.NewFake()
	fp := NewFakeProxier(ipt)
	svcIP := "172.30.0.41"
	svcPort := 80
	svcExternalIPs := "192.168.99.11"
	svcPortName := proxy.ServicePortName{
		NamespacedName: makeNSN("ns1", "svc1"),
		Port:           "p80",
	}

	makeServiceMap(fp,
		makeTestService(svcPortName.Namespace, svcPortName.Name, func(svc *v1.Service) {
			svc.Spec.Type = "NodePort"
			svc.Spec.ExternalTrafficPolicy = v1.ServiceExternalTrafficPolicyTypeLocal
			svc.Spec.ClusterIP = svcIP
			svc.Spec.ExternalIPs = []string{svcExternalIPs}
			svc.Spec.Ports = []v1.ServicePort{{
				Name:       svcPortName.Port,
				Port:       int32(svcPort),
				Protocol:   v1.ProtocolTCP,
				TargetPort: intstr.FromInt(svcPort),
			}}
		}),
	)
	epIP1 := "10.180.0.1"
	epIP2 := "10.180.2.1"
	tcpProtocol := v1.ProtocolTCP
	populateEndpointSlices(fp,
		makeTestEndpointSlice(svcPortName.Namespace, svcPortName.Name, 1, func(eps *discovery.EndpointSlice) {
			eps.AddressType = discovery.AddressTypeIPv4
			eps.Endpoints = []discovery.Endpoint{
				{
					Addresses: []string{epIP1},
				},
				{
					Addresses: []string{epIP2},
					NodeName:  utilpointer.StringPtr(testHostname),
				},
			}
			eps.Ports = []discovery.EndpointPort{{
				Name:     utilpointer.StringPtr(svcPortName.Port),
				Port:     utilpointer.Int32(int32(svcPort)),
				Protocol: &tcpProtocol,
			}}
		}),
	)

	fp.syncProxyRules()

	expected := dedent.Dedent(`
		*filter
		:KUBE-EXTERNAL-SERVICES - [0:0]
		:KUBE-FORWARD - [0:0]
		:KUBE-NODEPORTS - [0:0]
		:KUBE-SERVICES - [0:0]
		-A KUBE-FORWARD -m conntrack --ctstate INVALID -j DROP
		-A KUBE-FORWARD -m comment --comment "kubernetes forwarding rules" -m mark --mark 0x4000/0x4000 -j ACCEPT
		-A KUBE-FORWARD -m comment --comment "kubernetes forwarding conntrack rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
		COMMIT
		*nat
		:KUBE-EXT-XPGD46QRK7WJZT7O - [0:0]
		:KUBE-MARK-MASQ - [0:0]
		:KUBE-NODEPORTS - [0:0]
		:KUBE-POSTROUTING - [0:0]
		:KUBE-SEP-SXIVWICOYRO3J4NJ - [0:0]
		:KUBE-SEP-ZX7GRIZKSNUQ3LAJ - [0:0]
		:KUBE-SERVICES - [0:0]
		:KUBE-SVC-XPGD46QRK7WJZT7O - [0:0]
		:KUBE-SVL-XPGD46QRK7WJZT7O - [0:0]
		-A KUBE-SERVICES -m comment --comment "ns1/svc1:p80 cluster IP" -m tcp -p tcp -d 172.30.0.41 --dport 80 -j KUBE-SVC-XPGD46QRK7WJZT7O
		-A KUBE-SERVICES -m comment --comment "ns1/svc1:p80 external IP" -m tcp -p tcp -d 192.168.99.11 --dport 80 -j KUBE-EXT-XPGD46QRK7WJZT7O
		-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
		-A KUBE-EXT-XPGD46QRK7WJZT7O -m comment --comment "pod traffic for ns1/svc1:p80 external destinations" -s 10.0.0.0/8 -j KUBE-SVC-XPGD46QRK7WJZT7O
		-A KUBE-EXT-XPGD46QRK7WJZT7O -m comment --comment "masquerade LOCAL traffic for ns1/svc1:p80 external destinations" -m addrtype --src-type LOCAL -j KUBE-MARK-MASQ
		-A KUBE-EXT-XPGD46QRK7WJZT7O -m comment --comment "route LOCAL traffic for ns1/svc1:p80 external destinations" -m addrtype --src-type LOCAL -j KUBE-SVC-XPGD46QRK7WJZT7O
		-A KUBE-EXT-XPGD46QRK7WJZT7O -j KUBE-SVL-XPGD46QRK7WJZT7O
		-A KUBE-MARK-MASQ -j MARK --or-mark 0x4000
		-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
		-A KUBE-POSTROUTING -j MARK --xor-mark 0x4000
		-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE
		-A KUBE-SEP-SXIVWICOYRO3J4NJ -m comment --comment ns1/svc1:p80 -s 10.180.0.1 -j KUBE-MARK-MASQ
		-A KUBE-SEP-SXIVWICOYRO3J4NJ -m comment --comment ns1/svc1:p80 -m tcp -p tcp -j DNAT --to-destination 10.180.0.1:80
		-A KUBE-SEP-ZX7GRIZKSNUQ3LAJ -m comment --comment ns1/svc1:p80 -s 10.180.2.1 -j KUBE-MARK-MASQ
		-A KUBE-SEP-ZX7GRIZKSNUQ3LAJ -m comment --comment ns1/svc1:p80 -m tcp -p tcp -j DNAT --to-destination 10.180.2.1:80
		-A KUBE-SVC-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 cluster IP" -m tcp -p tcp -d 172.30.0.41 --dport 80 ! -s 10.0.0.0/8 -j KUBE-MARK-MASQ
		-A KUBE-SVC-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 -> 10.180.0.1:80" -m statistic --mode random --probability 0.5000000000 -j KUBE-SEP-SXIVWICOYRO3J4NJ
		-A KUBE-SVC-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 -> 10.180.2.1:80" -j KUBE-SEP-ZX7GRIZKSNUQ3LAJ
		-A KUBE-SVL-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 -> 10.180.2.1:80" -j KUBE-SEP-ZX7GRIZKSNUQ3LAJ
		COMMIT
		`)
	assertIPTablesRulesEqual(t, getLine(), expected, fp.iptablesData.String())

	runPacketFlowTests(t, getLine(), fp.iptablesData.String(), testNodeIP, []packetFlowTest{
		{
			name:     "cluster IP hits both endpoints",
			sourceIP: "10.0.0.2",
			destIP:   svcIP,
			destPort: svcPort,
			output:   fmt.Sprintf("%s:%d, %s:%d", epIP1, svcPort, epIP2, svcPort),
			masq:     false,
		},
		{
			name:     "external IP hits only local endpoint, unmasqueraded",
			sourceIP: testExternalClient,
			destIP:   svcExternalIPs,
			destPort: svcPort,
			output:   fmt.Sprintf("%s:%d", epIP2, svcPort),
			masq:     false,
		},
	})
}

// TestNonLocalExternalIPs tests if we add the masquerade rule into svcChain in order to
// SNAT packets to external IPs if externalTrafficPolicy is cluster and the traffic is NOT Local.
func TestNonLocalExternalIPs(t *testing.T) {
	ipt := iptablestest.NewFake()
	fp := NewFakeProxier(ipt)
	svcIP := "172.30.0.41"
	svcPort := 80
	svcExternalIPs := "192.168.99.11"
	svcPortName := proxy.ServicePortName{
		NamespacedName: makeNSN("ns1", "svc1"),
		Port:           "p80",
	}

	makeServiceMap(fp,
		makeTestService(svcPortName.Namespace, svcPortName.Name, func(svc *v1.Service) {
			svc.Spec.ClusterIP = svcIP
			svc.Spec.ExternalIPs = []string{svcExternalIPs}
			svc.Spec.Ports = []v1.ServicePort{{
				Name:       svcPortName.Port,
				Port:       int32(svcPort),
				Protocol:   v1.ProtocolTCP,
				TargetPort: intstr.FromInt(svcPort),
			}}
		}),
	)
	epIP1 := "10.180.0.1"
	epIP2 := "10.180.2.1"
	tcpProtocol := v1.ProtocolTCP
	populateEndpointSlices(fp,
		makeTestEndpointSlice(svcPortName.Namespace, svcPortName.Name, 1, func(eps *discovery.EndpointSlice) {
			eps.AddressType = discovery.AddressTypeIPv4
			eps.Endpoints = []discovery.Endpoint{{
				Addresses: []string{epIP1},
				NodeName:  nil,
			}, {
				Addresses: []string{epIP2},
				NodeName:  utilpointer.StringPtr(testHostname),
			}}
			eps.Ports = []discovery.EndpointPort{{
				Name:     utilpointer.StringPtr(svcPortName.Port),
				Port:     utilpointer.Int32(int32(svcPort)),
				Protocol: &tcpProtocol,
			}}
		}),
	)

	fp.syncProxyRules()

	expected := dedent.Dedent(`
		*filter
		:KUBE-EXTERNAL-SERVICES - [0:0]
		:KUBE-FORWARD - [0:0]
		:KUBE-NODEPORTS - [0:0]
		:KUBE-SERVICES - [0:0]
		-A KUBE-FORWARD -m conntrack --ctstate INVALID -j DROP
		-A KUBE-FORWARD -m comment --comment "kubernetes forwarding rules" -m mark --mark 0x4000/0x4000 -j ACCEPT
		-A KUBE-FORWARD -m comment --comment "kubernetes forwarding conntrack rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
		COMMIT
		*nat
		:KUBE-EXT-XPGD46QRK7WJZT7O - [0:0]
		:KUBE-MARK-MASQ - [0:0]
		:KUBE-NODEPORTS - [0:0]
		:KUBE-POSTROUTING - [0:0]
		:KUBE-SEP-SXIVWICOYRO3J4NJ - [0:0]
		:KUBE-SEP-ZX7GRIZKSNUQ3LAJ - [0:0]
		:KUBE-SERVICES - [0:0]
		:KUBE-SVC-XPGD46QRK7WJZT7O - [0:0]
		-A KUBE-SERVICES -m comment --comment "ns1/svc1:p80 cluster IP" -m tcp -p tcp -d 172.30.0.41 --dport 80 -j KUBE-SVC-XPGD46QRK7WJZT7O
		-A KUBE-SERVICES -m comment --comment "ns1/svc1:p80 external IP" -m tcp -p tcp -d 192.168.99.11 --dport 80 -j KUBE-EXT-XPGD46QRK7WJZT7O
		-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
		-A KUBE-EXT-XPGD46QRK7WJZT7O -m comment --comment "masquerade traffic for ns1/svc1:p80 external destinations" -j KUBE-MARK-MASQ
		-A KUBE-EXT-XPGD46QRK7WJZT7O -j KUBE-SVC-XPGD46QRK7WJZT7O
		-A KUBE-MARK-MASQ -j MARK --or-mark 0x4000
		-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
		-A KUBE-POSTROUTING -j MARK --xor-mark 0x4000
		-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE
		-A KUBE-SEP-SXIVWICOYRO3J4NJ -m comment --comment ns1/svc1:p80 -s 10.180.0.1 -j KUBE-MARK-MASQ
		-A KUBE-SEP-SXIVWICOYRO3J4NJ -m comment --comment ns1/svc1:p80 -m tcp -p tcp -j DNAT --to-destination 10.180.0.1:80
		-A KUBE-SEP-ZX7GRIZKSNUQ3LAJ -m comment --comment ns1/svc1:p80 -s 10.180.2.1 -j KUBE-MARK-MASQ
		-A KUBE-SEP-ZX7GRIZKSNUQ3LAJ -m comment --comment ns1/svc1:p80 -m tcp -p tcp -j DNAT --to-destination 10.180.2.1:80
		-A KUBE-SVC-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 cluster IP" -m tcp -p tcp -d 172.30.0.41 --dport 80 ! -s 10.0.0.0/8 -j KUBE-MARK-MASQ
		-A KUBE-SVC-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 -> 10.180.0.1:80" -m statistic --mode random --probability 0.5000000000 -j KUBE-SEP-SXIVWICOYRO3J4NJ
		-A KUBE-SVC-XPGD46QRK7WJZT7O -m comment --comment "ns1/svc1:p80 -> 10.180.2.1:80" -j KUBE-SEP-ZX7GRIZKSNUQ3LAJ
		COMMIT
		`)
	assertIPTablesRulesEqual(t, getLine(), expected, fp.iptablesData.String())

	runPacketFlowTests(t, getLine(), fp.iptablesData.String(), testNodeIP, []packetFlowTest{
		{
			name:     "pod to cluster IP",
			sourceIP: "10.0.0.2",
			destIP:   svcIP,
			destPort: svcPort,
			output:   fmt.Sprintf("%s:%d, %s:%d", epIP1, svcPort, epIP2, svcPort),
			masq:     false,
		},
		{
			name:     "external to external IP",
			sourceIP: testExternalClient,
			destIP:   svcExternalIPs,
			destPort: svcPort,
			output:   fmt.Sprintf("%s:%d, %s:%d", epIP1, svcPort, epIP2, svcPort),
			masq:     true,
		},
	})
}

```

