# 🚀 Privilege Escalation & Post-Exploitation Lab

![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![Linux](https://img.shields.io/badge/OS-Linux-orange)
![Security](https://img.shields.io/badge/Type-Security%20Research-red)

> Una guía completa sobre **Escalada de Privilegios** en Linux: desde la enumeración inicial hasta la obtención de acceso root. Técnicas reales utilizadas en pentesting profesional.

---

## 📋 Tabla de Contenidos

- [Introducción](#introducción)
- [Herramientas Clave](#herramientas-clave)
- [Fase 1: Enumeración](#fase-1-enumeración)
- [Fase 2: Explotación](#fase-2-explotación)
- [Fase 3: Consolidación](#fase-3-consolidación)
- [Mitigación](#mitigación)
- [Recursos](#recursos)

---

## 🎯 Introducción

Este repositorio documenta el flujo completo de **Post-Explotación** en sistemas Linux, enfocado en cómo un atacante con acceso inicial limitado puede escalar privilegios hasta obtener control root.

**Escenario típico:**
```
Usuario limitado (www-data) → Enumeración → Credencial comprometida → Root shell
```

---

## 🛠️ Herramientas Clave

### Comandos de Enumeración

| Comando | Propósito | Output Esperado |
|---------|-----------|-----------------|
| `cat /etc/passwd \| grep -v nologin` | Listar usuarios reales | Usuarios con shell interactiva |
| `getcap -r / 2>/dev/null` | Buscar binarios con capacidades | `cap_setuid+ep` en binarios peligrosos |
| `sudo -l` | Ver permisos sudo del usuario | Comandos ejecutables como root |
| `id` | Verificar identidad actual | `uid=0(root)` = Éxito |

---

## 🔍 Fase 1: Enumeración

### 1.1 Identificar Usuarios Reales

```bash
cat /etc/passwd | grep -v nologin
```

**¿Qué hace?**
- Lee el archivo `/etc/passwd` (base de datos de usuarios)
- Filtra con `grep -v nologin` para excluir cuentas de servicio
- Muestra solo usuarios con acceso interactivo

**Ejemplo de salida:**
```
root:x:0:0:root:/root:/bin/bash
kali:x:1000:1000:Kali User:/home/kali:/bin/bash
developer:x:1001:1001:Dev User:/home/developer:/bin/bash
```

**En pentesting:** Identifica objetivos potenciales para ataques de fuerza bruta o búsqueda de credenciales en archivos de configuración.

---

### 1.2 Búsqueda de Capabilities Peligrosas

```bash
getcap -r / 2>/dev/null
```

**¿Qué hace?**
- `getcap -r /`: Busca recursivamente desde raíz todos los archivos con capacidades especiales
- `2>/dev/null`: Redirige errores al "agujero negro" para limpiar la salida

**Ejemplo de salida peligrosa:**
```
/usr/bin/python3.9 = cap_setuid+ep
/usr/bin/perl = cap_setuid+ep
/usr/bin/tar = cap_setuid+ep
```

**Riesgo:** Estos binarios pueden usarse para generar una shell root instantáneamente usando técnicas de GTFOBins.

---

## ⚡ Fase 2: Explotación

### 2.1 Bypass de Sudo sin TTY Interactiva

**Problema:** En una reverse shell, `sudo` requiere una terminal interactiva para pedir contraseña.

**Solución:** Usar `sudo -S` para leer la contraseña desde stdin:

```bash
echo 'password' | sudo -S whoami
```

**Desglose:**
- `echo 'password'`: Genera la contraseña
- `|`: Pipe (envía salida al siguiente comando)
- `sudo -S`: Lee contraseña desde stdin en lugar de teclado
- `whoami`: Comando a ejecutar (verifica si somos root)

**Salida exitosa:**
```
root
```

---

### 2.2 Verificación de Escalada

```bash
echo 'password' | sudo -S id && echo 'password' | sudo -S whoami
```

**Output esperado:**
```
uid=0(root) gid=0(root) grupos=0(root)
root
```

**Interpretación:**
- `uid=0` = Usuario root (máximo privilegio)
- `gid=0` = Grupo root
- `whoami = root` = Confirmación adicional

---

## 🎖️ Fase 3: Consolidación

### Obtener Shell Completa de Root

```bash
echo 'password' | sudo -S /bin/bash
```

O para una sesión más limpia:

```bash
echo 'password' | sudo -S su -
```

**Ventajas:**
- No necesitas escribir `sudo` en cada comando
- Control total y permanente del sistema
- Acceso a archivos sensibles (`/root`, `/etc/shadow`, etc.)

---

## 🔴 Caso de Estudio: "El Bypass de Kali"

### Escenario Real

Un atacante obtiene acceso inicial a una máquina Kali Linux con el usuario `kali` y descubre que:

1. **Usuario:** `kali`
2. **Contraseña:** `kali` (débil)
3. **Permisos:** Tiene acceso a `sudo` sin restricciones

### Ejecución del Ataque

```bash
# Paso 1: Intento inicial (falla)
sudo cat /etc/sudoers.d/kali-grant-root
# Error: a terminal is required to read the password

# Paso 2: Bypass usando STDIN
echo 'kali' | sudo -S whoami
# root

# Paso 3: Verificación
echo 'kali' | sudo -S id
# uid=0(root) gid=0(root) grupos=0(root)

# Paso 4: Shell permanente
echo 'kali' | sudo -S su -
# root@kali:~#
```

### Resultado
✅ **Escalada exitosa de `kali` a `root`**

---

## 🛡️ Mitigación y Defensa

### Para Administradores

#### 1. Principio de Menor Privilegio
```bash
# ❌ MAL: Permitir todo
kali ALL=(ALL) ALL

# ✅ BIEN: Permitir solo comandos específicos
kali ALL=(ALL) /usr/bin/apt-get, /usr/bin/systemctl
```

#### 2. Requerir Contraseña en Sudo
```bash
# ❌ MAL
kali ALL=(ALL) NOPASSWD: ALL

# ✅ BIEN
kali ALL=(ALL) ALL  # Requiere contraseña
```

#### 3. Auditar Capabilities
```bash
# Revisar regularmente
getcap -r / 2>/dev/null

# Remover capabilities innecesarias
sudo setcap -r /usr/bin/python3.9
```

#### 4. Monitoreo
```bash
# Registrar intentos de sudo
sudo tail -f /var/log/auth.log | grep sudo
```

---

## 📚 Recursos Adicionales

### Herramientas Recomendadas
- **[GTFOBins](https://gtfobins.github.io/)** - Técnicas de escape desde binarios
- **[LinPEAS](https://github.com/carlospolop/PEASS-ng)** - Script de enumeración automática
- **[Sudo Exploits](https://www.exploit-db.com/)** - Base de datos de vulnerabilidades

### Certificaciones Relacionadas
- 🎓 OSCP (Offensive Security Certified Professional)
- 🎓 CEH (Certified Ethical Hacker)
- 🎓 GPEN (GIAC Penetration Tester)

### Plataformas de Práctica
- 🎮 [HackTheBox](https://www.hackthebox.com/)
- 🎮 [TryHackMe](https://www.tryhackme.com/)
- 🎮 [OverTheWire](https://overthewire.org/)

---

## ⚠️ Descargo de Responsabilidad

```
Este material es ÚNICAMENTE para propósitos educativos y de seguridad ofensiva ética.

✅ PERMITIDO:
   • Pruebas en sistemas propios
   • Laboratorios autorizados (HackTheBox, TryHackMe)
   • Auditorías de seguridad con consentimiento escrito

❌ PROHIBIDO:
   • Acceso no autorizado a sistemas
   • Daño intencional de infraestructura
   • Robo de datos o información sensible

El autor no se responsabiliza por el uso indebido de esta información.
```

---

## 📊 Flujo de Trabajo Completo

```
┌─────────────────────────────────────────────────────────┐
│  1. ACCESO INICIAL (ej: www-data, ssh débil)            │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  2. ENUMERACIÓN (getcap, /etc/passwd, sudo -l)          │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  3. BÚSQUEDA DE CREDENCIALES (archivos, historial)      │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  4. EXPLOTACIÓN (sudo -S, capabilities, SUID)           │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  5. CONSOLIDACIÓN (Shell root, persistencia)            │
└─────────────────────────────────────────────────────────┘
```

---

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor:

1. Fork el repositorio
2. Crea una rama (`git checkout -b feature/nueva-tecnica`)
3. Commit tus cambios (`git commit -am 'Agrega nueva técnica'`)
4. Push a la rama (`git push origin feature/nueva-tecnica`)
5. Abre un Pull Request

---

## 📝 Licencia

Este proyecto está bajo licencia **MIT**. Ver archivo `LICENSE` para más detalles.

---

## 👨‍💻 Autor

Creado con ❤️ para la comunidad de seguridad ofensiva.

**Última actualización:** 2024  
**Versión:** 2.0

---

> 💡 **Tip:** Guarda este repositorio en favoritos y revisalo regularmente. Las técnicas de escalada de privilegios evolucionan constantemente.
