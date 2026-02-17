
# HQ – Router0: IPsec VPN कॉन्फिगरेशन (Marathi स्पष्टीकरणांसह)

हा दस्तऐवज **HQ – Router0** वरील IPsec VPN कॉन्फिगरेशनचे सर्व commands तपशीलवार **मराठीत** समजावून सांगतो. प्रत्येक ओळीचा **का/कशासाठी** वापर केला जातो, त्यासोबत **अतिरिक्त टिप्स, पडताळणी (verification) commands, troubleshooting**, आणि **best practices** देखील दिल्या आहेत.

---

## 📦 आढावा (Overview)
- **उद्देश:** HQ (192.168.3.0/24) आणि Branch (192.168.4.0/24) यांच्यात **Site-to-Site IPsec VPN** उभारणे.
- **IKE Phase 1:** Peer ओळख आणि सुरक्षित negotiation (ISAKMP/IKEv1 किंवा IKEv2)
- **IPsec Phase 2:** Actual डेटा encryption/integrity (ESP)
- **ट्रिगर (Interesting Traffic):** ACL ने परिभाषित केलेले subnet-to-subnet ट्रॅफिक

> **टीप:** खालील उदाहरण IKEv1 syntax वापरते (ISAKMP शब्द दिसतो). IKEv2 साठी commands वेगळे असतात.

---

## 🗺️ सूचित टोपोलॉजी (Suggested Topology)
```
[HQ LAN] 192.168.3.0/24 --- (HQ Router0) ---[WAN/Internet]--- (Branch Router) --- 192.168.4.0/24 [BR LAN]

HQ Router Outside (WAN) → <public/WAN IP>
Branch Router Outside (WAN) → 192.168.2.2 (उदा.)
```

---

## 🔧 कॉन्फिगरेशन ब्लॉक (दिलेला)
```bash
access-list 100 permit ip 192.168.3.0 0.0.0.255 192.168.4.0 0.0.0.255
crypto isakmp policy 10
authentication pre-share
crypto isakmp key Pass@123 address 192.168.2.2
crypto ipsec transform-set HQ->Br1 esp-aes 256 esp-sha-hmac
crypto map IPSEC-MAP 10 ipsec-isakmp
 set peer 192.168.2.2
 set transform-set HQ->Br1
 match address 100
interface GigabitEthernet0/0/0
 crypto map IPSEC-MAP
```

---

## 🧩 प्रत्येक ओळीचे विस्तृत स्पष्टीकरण

### 1) `access-list 100 permit ip 192.168.3.0 0.0.0.255 192.168.4.0 0.0.0.255`
- **हे काय?**: VPN साठी **Crypto ACL** – कोणते ट्रॅफिक एन्क्रिप्ट करायचे ते ठरवते.
- **का आवश्यक?**: "Interesting traffic" ओळखले की VPN Phase 2 (IPsec) SAs तयार होतात.
- **अर्थ:**
  - Local subnet: `192.168.3.0/24`
  - Remote subnet: `192.168.4.0/24`
  - या दोन्ही subnet मधील **सर्व IP-to-IP** ट्रॅफिक एन्क्रिप्ट करा.
- **अतिरिक्त टिप्स:**
  - Branch बाजूला **मिरर ACL** असणे बंधनकारक (source/destination उलट).
  - Overlapping/चुकीची mask असल्यास tunnel येत नाही किंवा **proxy-ID mismatch** येतो.

### 2) `crypto isakmp policy 10`
- **हे काय?**: **IKE Phase 1** चे negotiation parameters define करण्यासाठी policy क्रमांक 10 तयार करतो.
- **का आवश्यक?**: दोन्ही बाजूंनी एकसारखे (किंवा compatible) parameters नसतील तर IKE Phase 1 **fail** होतो.
- **सामान्य parameters (IKEv1):** `encr`, `hash`, `authentication`, `group` (DH), `lifetime`. (इथे उदाहरणात defaults लागू असू शकतात.)
- **टीप:** अनेक policies असतील तर device सर्वोच्च प्राधान्यक्रमाने (lowest number) negotiate करतो.

### 3) `authentication pre-share`
- **हे काय?**: IKE Phase 1 साठी **Pre-Shared Key (PSK)** वापरणार.
- **का आवश्यक?**: Peer ची ओळख पटवण्यासाठी. दोन्हीकडे **एकच key** असणे आवश्यक.
- **सुरक्षा टिप:** मजबूत, जटिल PSK ठेवा आणि वारंवार बदलण्यासाठी प्रक्रिया ठेवा.

### 4) `crypto isakmp key Pass@123 address 192.168.2.2`
- **हे काय?**: **Peer 192.168.2.2** (Branch) साठी PSK `Pass@123` सेट करतो.
- **का आवश्यक?**: IKE Phase 1 authentication यशस्वी होण्यासाठी.
- **टीप:**
  - शक्यतो **public peer IP** वापरावा (NAT मागे असल्यास NAT‑Traversal आवश्यक).
  - PSK कॉन्फिग न जुळल्यास `authentication failed` logs दिसतात.

### 5) `crypto ipsec transform-set HQ->Br1 esp-aes 256 esp-sha-hmac`
- **हे काय?**: **IPsec Phase 2** चे security parameters.
- **अर्थ:**
  - **Encryption:** AES‑256 (मजबूत)
  - **Integrity/Auth:** SHA (HMAC)
- **का आवश्यक?**: डेटा एन्क्रिप्शन आणि tamper‑proofing.
- **टीप:** दोन्ही बाजूंना transform-set/proposal **जुळला पाहिजे**; mismatch असल्यास Phase 2 up होत नाही.

### 6) `crypto map IPSEC-MAP 10 ipsec-isakmp`
- **हे काय?**: `IPSEC-MAP` नावाचा **Crypto Map** sequence 10 तयार करतो (ISAKMP सह).
- **का आवश्यक?**: Peer, ACL आणि transform-set हे सर्व एकत्र **binding** करण्यासाठी.
- **टीप:** एकापेक्षा जास्त peers/ACLs असल्यास वेगवेगळे sequence numbers वापरा.

### 7) `set peer 192.168.2.2`
- **हे काय?**: VPN साठी **remote peer IP** परिभाषित करते.
- **का आवश्यक?**: कोणीाशी IKE negotiate करायचे ते कळते.
- **टीप:** जर peer dynamic असेल तर **Dynamic crypto map**/EZVPN इ. विचारात घ्या.

### 8) `set transform-set HQ->Br1`
- **हे काय?**: Crypto map वर Phase 2 transform-set **लागू** करतो.
- **का आवश्यक?**: map ला कोणते एन्क्रिप्शन/इंटिग्रिटी वापरायचे ते समजते.

### 9) `match address 100`
- **हे काय?**: Crypto map ला सांगते की कोणते ट्रॅफिक **encrypt** करायचे – ते ACL 100 मध्ये define आहे.
- **टीप:** Policy-based VPN मध्ये **याच match** मुळे tunnel ट्रिगर होतो. (Route-based/VTI मध्ये वेगळे.)

### 10) `interface GigabitEthernet0/0/0`  +  `crypto map IPSEC-MAP`
- **हे काय?**: हा **WAN/Outside इंटरफेस** गृहित धरला आहे. यावर crypto map **apply** केला आहे.
- **का आवश्यक?**: Interface वर crypto map नसल्यास IPsec कधीच लागू होत नाही.
- **टीप:** योग्य इंटरफेस निवडणे अत्यंत महत्वाचे; चुकीच्या इंटरफेसवर लावल्यास काहीच होणार नाही.

---

## ✅ पडताळणी (Verification) Commands – IOS/IOS‑XE

> खालील commands वापरून tunnel **up/down**, counters, आणि selectors तपासा.

```bash
show crypto isakmp sa              # IKEv1 Phase 1 स्थिती (MM_ACTIVE, QM_IDLE)
show crypto isakmp policy          # IKE policies
show crypto ikev2 sa               # (जर IKEv2 वापरत असाल तर)
show crypto ipsec sa               # Phase 2 SAs आणि packet counters
show crypto ipsec sa peer 192.168.2.2
show crypto map                    # crypto map binding तपासा
show run | section crypto map      # run-config मधील crypto map भाग
show access-lists 100              # Crypto ACL तपासा
show ip route 192.168.2.2          # Peer पर्यंत route
ping 192.168.4.10 source 192.168.3.10   # Interesting traffic ने tunnel trigger होते का?
traceroute 192.168.2.2 source <WAN_IP>  # Peer reachability
```

**काय बघायचे?**
- `show crypto isakmp sa` → `MM_ACTIVE` (Phase 1 पूर्ण), `QM_IDLE` (Phase 2 तयार)
- `show crypto ipsec sa` → **SPI दिसणे**, `#pkts encaps/decaps` **वाढणे** (टेस्ट करताना)
- **Local/Remote proxy IDs** (selectors) अपेक्षित subnet दाखवत आहेत का?

---

## 🧪 टेस्टिंग आणि Troubleshooting

### 1) Reachability
```bash
ping 192.168.2.2 source <WAN_Interface_IP>
```
- Peer reachable नसेल तर IKE Phase 1 सुरूच होणार नाही.

### 2) NAT Exemption (कॉमन issue)
- Policy-based VPN मध्ये interesting traffic वर **NAT लागू होऊ नये**.
```bash
show run | section nat
```
- जर NAT लागू असेल, tunnel येत नाही किंवा traffic decrypt होत नाही.

### 3) Debugs (काळजीपूर्वक)
```bash
debug crypto isakmp
debug crypto ipsec
terminal monitor
undebug all
show logging
```
- **Off-hours** मध्ये वापरा; CPU वर परिणाम होऊ शकतो.

### 4) Embedded Packet Capture (EPC)
```bash
monitor capture CAP interface <wan-int> match ip host <peer_ip> any
monitor capture CAP start
<traffic generate>
monitor capture CAP stop
show monitor capture CAP dump
no monitor capture CAP
```

### 5) सामान्य त्रुटी (Common Pitfalls)
- PSK mismatch → `authentication failed`
- ACL mismatch/overlap → `proxy mismatch` किंवा **Phase 2 up नाही**
- Crypto map **चुकीच्या इंटरफेसवर** → काहीच encrypt होत नाही
- NAT exemption नसणे → traffic plain/NAT होतो, tunnel जुळत नाही
- Routing चुकीचे → peer/remote subnet साठी योग्य route नाही

---

## 🔐 सुरक्षा Best Practices
- **मजबूत PSK** (लांब, random) आणि नियमित rotation
- शक्य असेल तेथे **IKEv2** वापरा (जास्त robust आणि secure)
- **AES‑GCM** सारखे आधुनिक proposals (IKEv2) विचारात घ्या
- **Perfect Forward Secrecy (PFS)** सक्षम करा (Phase 2 साठी DH group)
- **Logs/Monitoring**: Syslog/SNMP/NetFlow सह visibility ठेवा
- **Clock sync**: NTP कॉन्फिग (certificates किंवा logs साठी उपयुक्त)

---

## 🧱 Branch (उदाहरण) – मिररिंग संकल्पना
> हा फक्त **दिशादर्शक** नमुना आहे; तुमच्या branch च्या IPs नुसार बदल करा.
```bash
access-list 100 permit ip 192.168.4.0 0.0.0.255 192.168.3.0 0.0.0.255
crypto isakmp policy 10
authentication pre-share
crypto isakmp key Pass@123 address <HQ_WAN_IP>
crypto ipsec transform-set HQ->Br1 esp-aes 256 esp-sha-hmac
crypto map IPSEC-MAP 10 ipsec-isakmp
 set peer <HQ_WAN_IP>
 set transform-set HQ->Br1
 match address 100
interface GigabitEthernet0/0/0
 crypto map IPSEC-MAP
```

---

## 🗒️ जलद चेकलिस्ट (Up/Down Diagnosis)
- [ ] Peer reachable? (`ping/traceroute`)
- [ ] IKE Phase 1 up? (`show crypto isakmp sa` → `MM_ACTIVE`)
- [ ] Phase 2 up? (`show crypto ipsec sa` → SAs + counters)
- [ ] ACL/selectors mirrored?
- [ ] NAT exemption लागू?
- [ ] Crypto map योग्य इंटरफेसवर?
- [ ] Routes बरोबर?

---

## 📎 संदर्भ (General)
- Cisco IOS Security Command Reference (IPsec/IKEv1/IKEv2)
- Field अनुभव आणि मान्यताप्राप्त पद्धती (best practices)

---

*लेखक: तुमचा सहकारी – Pratik साठी खास तयार केलेले स्पष्टीकरण*
