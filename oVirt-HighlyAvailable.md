```diff 
- 這邊文章僅僅拿來記錄網路上所讀  
```
# 高可用性  
高可用性的用意在於解決電腦出狀況時的應對方法，較直接的方法就是為叢集下的各台電腦多做一台備援，當一台電腦出問題另外一台就可以直接使用。  

## oVirt high-avalibility  
如果要在oVirt上實現高可用性，首先就必須要有iDRAC or IPMI等設備，這兩種設備主要是管理主機的硬體設備，我認為比較重要的項目是電源管理的部分。  
還有一個要注意的點就是，如果要將出問題的主機資料移動到另一個目標上，則另一個目標必須擁有相同的實體資源，例如：  
第一台主機的實體資源  
```
CPU：4核
MEM：8G
```
那麼，第二台主機所擁有的實體資源也就要是  
```
CPU：4核
MEM：8G
```

在[2]的實驗中，將第一台主機的電源移除後，oVirt-engine 會立即確認被關閉的虛擬機並在第二台主機上開起虛擬機。  
如果 oVirt-engine 發現第一台主機可以重新通電並開機，那 oVirt-engine 就會相第一台主機開啟，並重新將其加入叢集裡。

## 如何在oVirt中開啟 Highly Available



#### 參考資料  
[1]Highly Available VMs：oVirt high avalhttps://www.ovirt.org/develop/ha-vms.html  
[2]Enabling High Availability Service with oVirt Virtualization and CephFS：https://indico.cern.ch/event/567550/papers/2627178/files/5648-ACAT2017_Proceedings_v3.1.pdf
