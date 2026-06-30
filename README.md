# 🔐 VPN Cisco IPSec IKEv1 Site-to-Site — Basada en Enrutamiento (VTI)

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![IPSec](https://img.shields.io/badge/IPSec-IKEv1-green?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-red?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y validación de una **VPN Cisco IPSec IKEv1 Site-to-Site punto a punto basada en enrutamiento**. A diferencia del modelo basado en políticas (crypto map + ACL), esta práctica utiliza una **interfaz virtual Tunnel0 (VTI)** protegida con IPSec, enrutando el tráfico entre las LANs remotas mediante **rutas estáticas** mediante el túnel.

---

## 🗺️ Topología de Red

La topología posee dos routers como peers VPN, un router ISP intermedio y una LAN en cada extremo. R1 representa el sitio A y R2 representa el sitio B.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP | Máscara | Función |
|:-----------:|:--------:|:------------:|:-------:|---------|
| R1 | Ethernet0/0 | 10.7.25.1 | /30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2 | /30 | Enlace hacia R1 |
| ISP | Ethernet0/1 | 10.7.25.6 | /30 | Enlace hacia R2 |
| R2 | Ethernet0/0 | 10.7.25.5 | /30 | WAN hacia ISP |
| R1 | Ethernet0/1 | 10.7.25.65 | /27 | Gateway LAN-A |
| VPC1 | eth0 | 10.7.25.66 | /27 | Cliente LAN-A |
| R2 | Ethernet0/1 | 10.7.25.97 | /27 | Gateway LAN-B |
| VPC2 | eth0 | 10.7.25.98 | /27 | Cliente LAN-B |
| **R1** | **Tunnel0** | **10.7.25.129** | **/30** | **VTI IPSec** |
| **R2** | **Tunnel0** | **10.7.25.130** | **/30** | **VTI IPSec** |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | Site-to-Site punto a punto |
| Modelo | **Route-Based VPN / VTI** |
| Interfaz virtual | Tunnel0 |
| Versión IKE | IKEv1 |
| Cifrado Fase 1 | AES 256 |
| Autenticación | Pre-Shared Key |
| Clave compartida | VPN12345 |
| Grupo Diffie-Hellman | Grupo 5 |
| Transform-set | TS-IKEV1 |
| IPSec | ESP-AES-256 + ESP-SHA-HMAC |
| PFS | Grupo 5 |
| Ruta R1 → LAN-B | 10.7.25.96/27 vía Tunnel0 |
| Ruta R2 → LAN-A | 10.7.25.64/27 vía Tunnel0 |

---

## 🚀 Scripts de Configuración

### R1 — Configuración Principal
```cisco
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
 group 5
crypto isakmp key VPN12345 address 10.7.25.5

crypto ipsec transform-set TS-IKEV1 esp-aes 256 esp-sha-hmac
 mode tunnel

crypto ipsec profile PROFILE-IKEV1
 set transform-set TS-IKEV1
 set pfs group5

interface Tunnel0
 description VTI_IPSEC_IKEV1_HACIA_R2
 ip address 10.7.25.129 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE-IKEV1

ip route 10.7.25.96 255.255.255.224 Tunnel0
```

### R2 — Configuración Principal
```cisco
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
 group 5
crypto isakmp key VPN12345 address 10.7.25.1

crypto ipsec transform-set TS-IKEV1 esp-aes 256 esp-sha-hmac
 mode tunnel

crypto ipsec profile PROFILE-IKEV1
 set transform-set TS-IKEV1
 set pfs group5

interface Tunnel0
 description VTI_IPSEC_IKEV1_HACIA_R1
 ip address 10.7.25.130 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE-IKEV1

ip route 10.7.25.64 255.255.255.224 Tunnel0
```

---

## ✅ Verificación del Túnel

```cisco
show interface tunnel0
show crypto isakmp sa
show crypto ipsec sa
show crypto session
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show interface tunnel0` | Tunnel0 up/up |
| `show crypto isakmp sa` | QM_IDLE / ACTIVE |
| `show crypto ipsec sa` | pkts encaps y decaps activos |
| `show crypto session` | UP-ACTIVE |

> 💡 **Nota técnica:** En `show crypto session` puede aparecer el origen como "crypto map" debido al funcionamiento interno de Cisco para interfaces VTI. Esto es normal y **no** significa que la VPN sea basada en políticas.

---

## 🔍 Diferencias: Basada en Políticas vs. Basada en Enrutamiento

| Elemento | VPN basada en políticas | VPN basada en enrutamiento |
|:--------:|:------------------------:|:----------------------------:|
| Método | Crypto map + ACL | Tunnel0 / VTI + rutas |
| Selección del tráfico | ACL de tráfico interesante | Tabla de enrutamiento |
| Aplicación IPSec | Crypto map en la WAN | Tunnel protection en Tunnel0 |
| Comprobación principal | `show crypto map` / ACL matches | `show interface tunnel0` / rutas vía Tunnel0 |

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Tunnel0 en R1 y R2 | ✅ up/up |
| Ruta hacia LAN remota | ✅ Vía Tunnel0 en ambos routers |
| IKEv1 SA | ✅ QM_IDLE / ACTIVE |
| IPSec SA | ✅ ACTIVE con encaps/decaps |
| Crypto session | ✅ UP-ACTIVE |
| Ping VPC1 → VPC2 | ✅ Exitoso |
| Ping VPC2 → VPC1 | ✅ Exitoso |

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" en el trace no es una falla — VPCS usa paquetes UDP, y al llegar al destino el host responde que el puerto UDP no está disponible. El trace confirma que el tráfico viaja por las IPs del túnel (10.7.25.129 / 10.7.25.130).

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_VPN_IPSec-IKEv1-Site-to-Site-basada-en-enrutamientoP2.txt`](SaelGerman_2025-0725_Script_VPN_IPSec-IKEv1-Site-to-Site-basada-en-enrutamientoP2.txt) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_VPN-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO_P2.pdf`](SaelGerman_2025-0725_VPN-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO_P2.pdf) | Documentación técnica completa |

---

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología general de la VPN](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%201.%20Topolog%C3%ADa%20general%20de%20la%20VPN%20Cisco%20IPSec%20IKEv1%20Site-to-Site%20basada%20en%20enrutamiento.png)
- 📸 [Figura 2 — R1: interfaces activas, incluyendo Tunnel0 con IP 10.7.25.129](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%202.%20R1%20interfaces%20activas%2C%20incluyendo%20Tunnel0%20con%20IP%2010.7.25.129.png)
- 📸 [Figura 3 — R1: configuración de la interfaz Tunnel0 como VTI IPSec](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%203.%20R1%20configuraci%C3%B3n%20de%20la%20interfaz%20Tunnel0%20como%20VTI%20IPSec.png)
- 📸 [Figura 4 — R1: configuración crypto IKEv1, transform-set y perfil IPSec](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%204.%20R1%20configuraci%C3%B3n%20crypto%20IKEv1%2C%20transform-set%20y%20perfil%20IPSec.png)
- 📸 [Figura 5 — R1: ruta estática hacia LAN-B usando Tunnel0](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%205.%20R1%20ruta%20est%C3%A1tica%20hacia%20LAN-B%20usando%20Tunnel0.png)
- 📸 [Figura 6 — R1: estado ISAKMP QM_IDLE / ACTIVE](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%206.%20R1%20estado%20ISAKMP%20QM_IDLE%20%20ACTIVE.png)
- 📸 [Figura 7 — R1: contadores IPSec con paquetes encapsulados y decapsulados](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%207.%20R1%20contadores%20IPSec%20con%20paquetes%20encapsulados%20y%20decapsulados.png)
- 📸 [Figura 8 — R1: Security Associations IPSec activas](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%208.%20R1%20Security%20Associations%20IPSec%20activas.png)
- 📸 [Figura 9 — R1: sesión crypto en estado UP-ACTIVE](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%209.%20R1%20sesi%C3%B3n%20crypto%20en%20estado%20UP-ACTIVE.png)
- 📸 [Figura 10 — VPC1: ping y trace hacia VPC2](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2010.%20VPC1%20ping%20y%20trace%20hacia%20VPC2.png)
- 📸 [Figura 11 — VPC2: ping y trace hacia VPC1](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2011.%20VPC2%20ping%20y%20trace%20hacia%20VPC1.png)
- 📸 [Figura 12 — R2: interfaces activas, incluyendo Tunnel0 con IP 10.7.25.130](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2012.%20R2%20interfaces%20activas%2C%20incluyendo%20Tunnel0%20con%20IP%2010.7.25.130.png)
- 📸 [Figura 13 — R2: configuración de la interfaz Tunnel0 como VTI IPSec](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2013.%20R2%20configuraci%C3%B3n%20de%20la%20interfaz%20Tunnel0%20como%20VTI%20IPSec.png)
- 📸 [Figura 14 — R2: configuración crypto IKEv1, transform-set y perfil IPSec](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2014.%20R2%20configuraci%C3%B3n%20crypto%20IKEv1%2C%20transform-set%20y%20perfil%20IPSec.png)
- 📸 [Figura 15 — R2: ruta estática hacia LAN-A usando Tunnel0](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2015.%20R2%20ruta%20est%C3%A1tica%20hacia%20LAN-A%20usando%20Tunnel0.png)
- 📸 [Figura 16 — R2: estado ISAKMP QM_IDLE / ACTIVE](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2016.%20R2%20estado%20ISAKMP%20QM_IDLE%20%20ACTIVE.png)
- 📸 [Figura 17 — R2: contadores IPSec con paquetes encapsulados y decapsulados](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2017.%20R2%20contadores%20IPSec%20con%20paquetes%20encapsulados%20y%20decapsulados.png)
- 📸 [Figura 18 — R2: Security Associations IPSec activas](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2018.%20R2%20Security%20Associations%20IPSec%20activas.png)
- 📸 [Figura 19 — R2: sesión crypto en estado UP-ACTIVE](VPN-CISCO-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO-capturas/Figura%2019.%20R2%20sesi%C3%B3n%20crypto%20en%20estado%20UP-ACTIVE.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_VPN-IPSEC-IKEv1-SITE-TO-SITE-BASADA-EN-ENRUTAMIENTO_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/hElXsqso-H0)
🔗 **VPN Basada en Políticas (relacionada):** [Ver Repositorio](https://github.com/Sael2727/Sael2727-VPN-Cisco-IPSec-IKEv1-Site-to-Site-basada-en-politicas.git)

---

## 📚 Referencias

1. Cisco Systems. *Configuring Virtual Tunnel Interfaces (VTI) for IPSec*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
