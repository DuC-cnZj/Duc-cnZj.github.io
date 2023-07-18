---
title: k8s api 兼容性历史
date: '2023-07-18 14:14:38'
sidebar: false
categories:
 - 技术
tags:
 - k8s
publish: true
---

## v1.20

### 启用的

```go
admissionregistrationv1.SchemeGroupVersion,
admissionregistrationv1beta1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authenticationv1beta1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
authorizationapiv1beta1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
certificatesapiv1beta1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
coordinationapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
extensionsapiv1beta1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
networkingapiv1beta1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion,
policyapiv1beta1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
rbacv1beta1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
schedulingapiv1beta1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,

extensionsapiv1beta1.SchemeGroupVersion.WithResource("ingresses"),
```

### 禁用的

```
apiserverinternalv1alpha1.SchemeGroupVersion,
batchapiv2alpha1.SchemeGroupVersion,
nodev1alpha1.SchemeGroupVersion,
rbacv1alpha1.SchemeGroupVersion,
schedulingv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```





## v1.21

### 启用的

```
admissionregistrationv1.SchemeGroupVersion,
admissionregistrationv1beta1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authenticationv1beta1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
authorizationapiv1beta1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
certificatesapiv1beta1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
coordinationapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
extensionsapiv1beta1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
networkingapiv1beta1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion,
policyapiv1beta1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
rbacv1beta1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
schedulingapiv1beta1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
extensionsapiv1beta1.SchemeGroupVersion.WithResource("ingresses"),
```

### 禁用的

```
apiserverinternalv1alpha1.SchemeGroupVersion,
nodev1alpha1.SchemeGroupVersion,
rbacv1alpha1.SchemeGroupVersion,
schedulingv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```





## v1.22

### 启用的

```
admissionregistrationv1.SchemeGroupVersion,
admissionregistrationv1beta1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authenticationv1beta1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
authorizationapiv1beta1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
certificatesapiv1beta1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
coordinationapiv1beta1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
extensionsapiv1beta1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
networkingapiv1beta1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
policyapiv1beta1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
rbacv1beta1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
schedulingapiv1beta1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
extensionsapiv1beta1.SchemeGroupVersion.WithResource("ingresses"),
```

### 禁用的

```
apiserverinternalv1alpha1.SchemeGroupVersion,
nodev1alpha1.SchemeGroupVersion,
rbacv1alpha1.SchemeGroupVersion,
schedulingv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```





## v1.23

## 启用

```
admissionregistrationv1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
policyapiv1beta1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,
flowcontrolv1beta2.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
```

### 禁用

```
apiserverinternalv1alpha1.SchemeGroupVersion,
nodev1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```



## v1.24

### 启用

```
admissionregistrationv1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,

autoscalingapiv2beta1.SchemeGroupVersion.WithResource("horizontalpodautoscalers"), // remove in 1.25
autoscalingapiv2beta2.SchemeGroupVersion.WithResource("horizontalpodautoscalers"), // remove in 1.26
batchapiv1beta1.SchemeGroupVersion.WithResource("cronjobs"),                       // remove in 1.25
discoveryv1beta1.SchemeGroupVersion.WithResource("endpointslices"),                // remove in 1.25
eventsv1beta1.SchemeGroupVersion.WithResource("events"),                           // remove in 1.25
nodev1beta1.SchemeGroupVersion.WithResource("runtimeclasses"),                     // remove in 1.25
policyapiv1beta1.SchemeGroupVersion.WithResource("poddisruptionbudgets"),          // remove in 1.25
policyapiv1beta1.SchemeGroupVersion.WithResource("podsecuritypolicies"),           // remove in 1.25
storageapiv1beta1.SchemeGroupVersion.WithResource("csinodes"),                     // remove in 1.25
storageapiv1beta1.SchemeGroupVersion.WithResource("csistoragecapacities"),         // remove in 1.27
flowcontrolv1beta1.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.26
flowcontrolv1beta1.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.26
flowcontrolv1beta2.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.29
flowcontrolv1beta2.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.29
```

### 禁用

```
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion, // remove in 1.26
policyapiv1beta1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
flowcontrolv1beta2.SchemeGroupVersion,
apiserverinternalv1alpha1.SchemeGroupVersion,
networkingapiv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```



## v1.25

### 启用

```
admissionregistrationv1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,

autoscalingapiv2beta2.SchemeGroupVersion.WithResource("horizontalpodautoscalers"), // remove in 1.26
storageapiv1beta1.SchemeGroupVersion.WithResource("csistoragecapacities"),         // remove in 1.27
flowcontrolv1beta1.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.26
flowcontrolv1beta1.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.26
flowcontrolv1beta2.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.29
flowcontrolv1beta2.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.29
```

### 禁用

```
apiserverinternalv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion, // remove in 1.26
policyapiv1beta1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
flowcontrolv1beta2.SchemeGroupVersion,
```





## v1.26

## 启用

```
admissionregistrationv1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,

autoscalingapiv2beta2.SchemeGroupVersion.WithResource("horizontalpodautoscalers"), // remove in 1.26
storageapiv1beta1.SchemeGroupVersion.WithResource("csistoragecapacities"),         // remove in 1.27
flowcontrolv1beta1.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.26
flowcontrolv1beta1.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.26
flowcontrolv1beta2.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.29
flowcontrolv1beta2.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.29
flowcontrolv1beta3.SchemeGroupVersion.WithResource("flowschemas"),                 // deprecate in 1.29, remove in 1.32
flowcontrolv1beta3.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // deprecate in 1.29, remove in 1.32
```

### 禁用

```
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion, // remove in 1.26
policyapiv1beta1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
flowcontrolv1beta2.SchemeGroupVersion,
flowcontrolv1beta3.SchemeGroupVersion,
admissionregistrationv1alpha1.SchemeGroupVersion,
apiserverinternalv1alpha1.SchemeGroupVersion,
authenticationv1alpha1.SchemeGroupVersion,
networkingapiv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```





## v1.27

### 启用

```
admissionregistrationv1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,

flowcontrolv1beta2.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.29
flowcontrolv1beta2.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.29
flowcontrolv1beta3.SchemeGroupVersion.WithResource("flowschemas"),                 // deprecate in 1.29, remove in 1.32
flowcontrolv1beta3.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // deprecate in 1.29, remove in 1.32
```

### 禁用

```
authenticationv1beta1.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion, // remove in 1.26
policyapiv1beta1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
flowcontrolv1beta2.SchemeGroupVersion,
flowcontrolv1beta3.SchemeGroupVersion,
admissionregistrationv1alpha1.SchemeGroupVersion,
apiserverinternalv1alpha1.SchemeGroupVersion,
authenticationv1alpha1.SchemeGroupVersion,
resourcev1alpha2.SchemeGroupVersion,
certificatesv1alpha1.SchemeGroupVersion,
networkingapiv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```





## v1.28

### 启用

```
admissionregistrationv1.SchemeGroupVersion,
apiv1.SchemeGroupVersion,
appsv1.SchemeGroupVersion,
authenticationv1.SchemeGroupVersion,
authorizationapiv1.SchemeGroupVersion,
autoscalingapiv1.SchemeGroupVersion,
autoscalingapiv2.SchemeGroupVersion,
batchapiv1.SchemeGroupVersion,
certificatesapiv1.SchemeGroupVersion,
coordinationapiv1.SchemeGroupVersion,
discoveryv1.SchemeGroupVersion,
eventsv1.SchemeGroupVersion,
networkingapiv1.SchemeGroupVersion,
nodev1.SchemeGroupVersion,
policyapiv1.SchemeGroupVersion,
rbacv1.SchemeGroupVersion,
storageapiv1.SchemeGroupVersion,
schedulingapiv1.SchemeGroupVersion,

flowcontrolv1beta2.SchemeGroupVersion.WithResource("flowschemas"),                 // remove in 1.29
flowcontrolv1beta2.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // remove in 1.29
flowcontrolv1beta3.SchemeGroupVersion.WithResource("flowschemas"),                 // deprecate in 1.29, remove in 1.32
flowcontrolv1beta3.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // deprecate in 1.29, remove in 1.32
```

### 禁用

```
authenticationv1beta1.SchemeGroupVersion,
autoscalingapiv2beta1.SchemeGroupVersion,
autoscalingapiv2beta2.SchemeGroupVersion,
batchapiv1beta1.SchemeGroupVersion,
discoveryv1beta1.SchemeGroupVersion,
eventsv1beta1.SchemeGroupVersion,
nodev1beta1.SchemeGroupVersion, // remove in 1.26
policyapiv1beta1.SchemeGroupVersion,
storageapiv1beta1.SchemeGroupVersion,
flowcontrolv1beta1.SchemeGroupVersion,
flowcontrolv1beta2.SchemeGroupVersion,
flowcontrolv1beta3.SchemeGroupVersion,
admissionregistrationv1alpha1.SchemeGroupVersion,
apiserverinternalv1alpha1.SchemeGroupVersion,
authenticationv1alpha1.SchemeGroupVersion,
resourcev1alpha2.SchemeGroupVersion,
certificatesv1alpha1.SchemeGroupVersion,
networkingapiv1alpha1.SchemeGroupVersion,
storageapiv1alpha1.SchemeGroupVersion,
flowcontrolv1alpha1.SchemeGroupVersion,
```

