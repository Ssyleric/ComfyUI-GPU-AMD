# README – GPU AMD (VAAPI) pour LXC ComfyUI (CT 21608188)

## 1. Contexte & objectif

Ce document décrit **comment exposer le GPU AMD Radeon RX 6700 XT** du serveur Proxmox VE vers le conteneur **ComfyUI LXC** (CT **21608188**, Debian 13), et comment **vérifier le bon fonctionnement de l’accélération matérielle vidéo (VAAPI)** à l’intérieur du conteneur.

> ⚠️ Important :
>
> - Ce README couvre l’exposition du GPU et les tests **VAAPI (décodage/encodage vidéo)**.
> - L’accélération **Stable Diffusion / IA de ComfyUI** côté AMD nécessite en plus une stack **ROCm + PyTorch ROCm**, non couverte ici.

---

## 2. Prérequis

- Hôte Proxmox VE 8 avec noyau :

```bash
uname -a
# Linux pve 6.8.12-16-pve ...
```

- GPU AMD visible sur l’hôte :

```bash
lspci -nnk -s 23:00.0
# 23:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. ...
# Kernel driver in use: amdgpu

ls -l /dev/kfd /dev/dri
# /dev/kfd
# /dev/dri/card0
# /dev/dri/renderD128
```

- LXC ComfyUI créé via **community-scripts** :
  - ID : **21608188**
  - Hostname : `comfyui`
  - OS : Debian 13 (trixie)

---

## 3. ETAPE 1 – Configuration du conteneur LXC ComfyUI

Éditer la configuration du CT côté **PVE**.

```bash
pct stop 21608188

nano /etc/pve/lxc/21608188.conf
```

Configuration **finale** utilisée (adapter uniquement cores/mémoire si besoin) :

```ini
arch: amd64
cmode: shell
cores: 4
features: keyctl=1,nesting=1
hostname: comfyui
memory: 32768
nameserver: 192.168.1.3
net0: name=eth0,bridge=vmbr0,gw=192.168.1.1,hwaddr=BC:24:11:40:40:37,ip=192.168.1.216/24,type=veth
onboot: 1
ostype: debian
rootfs: vm-docker:vm-21608188-disk-0,size=100G
searchdomain: z-server.me
startup: order=15
swap: 512
tags: ai;community-script
unprivileged: 1

# Exposition GPU AMD dans le CT
dev0: /dev/kfd,gid=993,uid=0
dev1: /dev/dri/renderD128,gid=44
```

Puis redémarrer le conteneur :

```bash
pct start 21608188
pct console 21608188
# login : root
```

---

## 4. ETAPE 2 – Vérifier le GPU dans le CT ComfyUI

Dans le conteneur :

```bash
ls -l /dev/kfd /dev/dri/renderD128
grep -w 'video\|render' /etc/group
```

Exemple attendu :

```text
crw-rw---- 1 root video 226, 128 ... /dev/dri/renderD128
crw-rw---- 1 root kvm   234,   0 ... /dev/kfd

video:x:44:root
render:x:992:root
```

👉 Ça confirme :

- que les deux devices sont bien **mappés** dans le CT,
- que l’utilisateur **root** peut les utiliser.

---

## 5. ETAPE 3 – Installer la stack VAAPI / Vulkan / FFmpeg dans le CT

Toujours dans le conteneur ComfyUI :

```bash
apt update
apt install -y vainfo mesa-va-drivers mesa-vulkan-drivers ffmpeg
```

---

## 6. ETAPE 4 – Test VAAPI avec `vainfo`

Dans le CT :

```bash
export LIBVA_DRIVER_NAME=radeonsi

vainfo --display drm --device /dev/dri/renderD128   | egrep 'Driver version|VAProfile'
```

Sortie attendue (exemple réel) :

```text
libva info: VA-API version 1.22.0
libva info: User environment variable requested driver 'radeonsi'
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
libva info: Found init function __vaDriverInit_1_22
libva info: va_openDriver() returns 0
vainfo: Driver version: Mesa Gallium driver 25.0.7-2 for AMD Radeon RX 6700 XT ...
vainfo: Supported profile and entrypoints
      VAProfileH264High               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileAV1Profile0            : VAEntrypointVLD
      ...
```

👉 Si tu vois bien :

- la **RX 6700 XT** dans la ligne *Driver version*,
- la liste des **VAProfile***,

alors **VAAPI est opérationnel dans le LXC ComfyUI**.

---

## 7. ETAPE 5 – Test d’encodage matériel avec FFmpeg (h264_vaapi)

Toujours dans le CT :

```bash
ffmpeg -v verbose   -init_hw_device vaapi=va:/dev/dri/renderD128   -filter_hw_device va   -f lavfi -i testsrc2=size=1920x1080:rate=30   -t 5   -vf 'format=nv12,hwupload'   -c:v h264_vaapi   -f null -
```

Points clés dans la sortie :

```text
[AVHWDeviceContext] Initialised VAAPI connection ...
[AVHWDeviceContext] VAAPI driver: Mesa Gallium driver ... for AMD Radeon RX 6700 XT ...
[h264_vaapi] Using VAAPI profile VAProfileH264High ...
[h264_vaapi] Using VAAPI entrypoint VAEntrypointEncSlice ...
frame=150 fps=... Lsize=N/A time=00:00:04.96 ...
video:7100KiB ...
```

👉 Si tu obtiens ces lignes avec 150 frames encodées et **aucune erreur**, l’encodage **H.264 via VAAPI** dans le LXC ComfyUI est **validé**.

---

## 8. Intégration avec ComfyUI (principes)

ComfyUI lui-même (pour Stable Diffusion) ne s’appuie pas sur VAAPI mais sur :

- CUDA (NVIDIA) ou
- ROCm / HIP (AMD) + PyTorch ROCm.

Ce README garantit que :

- le **GPU AMD est bien exposé** au conteneur ComfyUI,
- la **stack vidéo** (VAAPI / Mesa / FFmpeg) fonctionne.

Pour exploiter le GPU dans ComfyUI pour l’IA (diffusion, etc.), il faudra :

1. Installer une stack **ROCm** compatible Debian 13 / RDNA2.
2. Installer un **PyTorch ROCm** compatible avec cette version de ROCm.
3. Configurer ComfyUI pour utiliser ce backend (généralement via `torch.device("cuda")` / `"hip"` côté scripts).

---

## 9. Résumé rapide

- ✅ `/dev/kfd` & `/dev/dri/renderD128` mappés dans `21608188`
- ✅ `vainfo` voit la **RX 6700 XT** et les profils H264/HEVC/AV1
- ✅ `ffmpeg` encode en **h264_vaapi** depuis `testsrc2` sans erreur

Tu as donc un **LXC ComfyUI avec GPU AMD exposé et VAAPI fonctionnel**, prêt pour les prochains tests (ROCm / IA) ou pour traiter de la vidéo dans des workflows ComfyUI adaptés.
