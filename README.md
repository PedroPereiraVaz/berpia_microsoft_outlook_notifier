# Microsoft Outlook Token Notifier

## 📋 Índice

1. [Descripción General](#descripción-general)
2. [El Problema que Resuelve](#el-problema-que-resuelve)
3. [Arquitectura del Módulo](#arquitectura-del-módulo)
4. [Estructura de Archivos](#estructura-de-archivos)
5. [Modelos y Clases](#modelos-y-clases)
6. [Configuración del Sistema](#configuración-del-sistema)
7. [Dependencias](#dependencias)
8. [Flujo de Ejecución](#flujo-de-ejecución)
9. [Código Core vs No-Core](#código-core-vs-no-core)
10. [Instalación](#instalación)
11. [Configuración de Uso](#configuración-de-uso)
12. [Testing Manual](#testing-manual)
13. [Troubleshooting](#troubleshooting)
14. [Mantenimiento y Actualizaciones](#mantenimiento-y-actualizaciones)

---

## Descripción General

**Microsoft Outlook Token Notifier** es un módulo complementario para Odoo que monitorea la salud de las integraciones con Microsoft Outlook (Azure AD OAuth2) y notifica a los administradores cuando:

1. **El client secret está próximo a expirar** (30 días de preaviso)
2. **El client secret ha expirado o es inválido** (detección automática)

### Características Principales

| Característica            | Descripción                                              |
| ------------------------- | -------------------------------------------------------- |
| ⏰ Monitoreo automático   | Cron job diario que valida todos los servidores Outlook  |
| 📧 Notificación por email | Envía alertas a todos los usuarios administradores       |
| 💬 Notificación en Odoo   | Publica alertas en el canal de administradores (Discuss) |
| 🔄 Anti-spam              | Máximo 1 notificación por día                            |
| 📅 Preaviso configurable  | Campo para establecer fecha de expiración manualmente    |

---

## El Problema que Resuelve

### Contexto: Microsoft Azure AD OAuth2

Cuando configuras Outlook en Odoo, debes crear una **App Registration** en Azure Portal con:

- **Client ID** (Application ID) - Identificador de la aplicación
- **Client Secret** - Contraseña de la aplicación

**El problema:** Los Client Secrets de Azure AD tienen **fecha de expiración obligatoria** (6 meses, 12 meses, 24 meses, o personalizado). Cuando el secret expira:

```
❌ Los emails dejan de enviarse
❌ Los emails dejan de recibirse
❌ El error solo aparece en los logs del servidor
❌ El usuario final no recibe ninguna notificación
```

### Solución de Este Módulo

```
✅ Detecta automáticamente cuando el secret falla
✅ Notifica por email y Discuss
✅ Avisa 30 días ANTES de que expire (si configuras la fecha)
✅ Los administradores pueden actuar proactivamente
```

---

## Arquitectura del Módulo

### Diagrama de Flujo

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CRON JOB DIARIO                              │
│                  _cron_check_outlook_tokens()                        │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
         ┌─────────────────────────────────────────────┐
         │         ¿Ya notificó hoy?                   │
         │    (outlook_notifier_last_date)             │
         └─────────────────────────────────────────────┘
                    │                    │
                   SÍ                   NO
                    │                    │
                    ▼                    ▼
               ┌────────┐    ┌──────────────────────────────────┐
               │ SALIR  │    │  1. Verificar fecha expiración   │
               └────────┘    │     (microsoft_outlook_secret_   │
                             │      expiration)                 │
                             └──────────────────────────────────┘
                                            │
                                            ▼
                             ┌──────────────────────────────────┐
                             │  2. Validar tokens de:           │
                             │     - ir.mail_server (saliente)  │
                             │     - fetchmail.server (entrante)│
                             └──────────────────────────────────┘
                                            │
                                            ▼
                             ┌──────────────────────────────────┐
                             │  ¿Hay alertas que enviar?        │
                             └──────────────────────────────────┘
                                   │                │
                                  SÍ               NO
                                   │                │
                                   ▼                ▼
                    ┌────────────────────────┐    ┌────────┐
                    │ _send_notifications()  │    │ SALIR  │
                    │   - Email a admins     │    └────────┘
                    │   - Post en Discuss    │
                    │   - Guardar fecha      │
                    └────────────────────────┘
```

### Patrón de Diseño

El módulo sigue el patrón **"Scheduled Job"** de Odoo:

1. Un **ir.cron** (Scheduled Action) se ejecuta periódicamente
2. Llama a un método en un **AbstractModel** (modelo sin tabla en BD)
3. El método realiza las verificaciones y envía notificaciones

---

## Estructura de Archivos

```
berpia_microsoft_outlook_notifier/
│
├── __init__.py                 # Inicializador del módulo
├── __manifest__.py             # Metadatos y configuración del módulo
│
├── models/
│   ├── __init__.py             # Importa los modelos
│   └── outlook_notifier.py     # TODA la lógica del módulo (core file)
│
├── data/
│   └── ir_cron.xml             # Definición del job programado
│
├── views/
│   └── res_config_settings_views.xml  # UI en Settings
│
├── security/
│   └── ir.model.access.csv     # Permisos de acceso
│
└── README.md                   # Esta documentación
```

### Propósito de Cada Archivo

| Archivo                               | Tipo   | Propósito                                               |
| ------------------------------------- | ------ | ------------------------------------------------------- |
| `__init__.py` (raíz)                  | Python | Importa el paquete `models`                             |
| `__manifest__.py`                     | Python | Define nombre, versión, dependencias, archivos de datos |
| `models/__init__.py`                  | Python | Importa `outlook_notifier`                              |
| `models/outlook_notifier.py`          | Python | **ARCHIVO PRINCIPAL** - Contiene toda la lógica         |
| `data/ir_cron.xml`                    | XML    | Define el cron job que se ejecuta diariamente           |
| `views/res_config_settings_views.xml` | XML    | Añade campo de fecha en Settings → Outlook              |
| `security/ir.model.access.csv`        | CSV    | Da permisos de lectura al modelo                        |

---

## Modelos y Clases

### Archivo: `models/outlook_notifier.py`

Este archivo contiene **dos clases**:

---

### Clase 1: `ResConfigSettings`

```python
class ResConfigSettings(models.TransientModel):
    _inherit = 'res.config.settings'
```

**Propósito:** Añadir un campo de fecha en la configuración general de Odoo.

**Tipo de Modelo:** `TransientModel` (modelo temporal, no persiste en BD directamente)

**Herencia:** Extiende `res.config.settings` (el modelo de configuración general de Odoo)

#### Campos

| Campo                                 | Tipo   | Descripción                           |
| ------------------------------------- | ------ | ------------------------------------- |
| `microsoft_outlook_secret_expiration` | `Date` | Fecha de expiración del client secret |

#### Métodos

##### `get_values(self)`

```python
@api.model
def get_values(self):
    res = super().get_values()
    param = self.env['ir.config_parameter'].sudo().get_param(
        'microsoft_outlook_secret_expiration', ''
    )
    if param:
        try:
            res['microsoft_outlook_secret_expiration'] = fields.Date.from_string(param)
        except (ValueError, TypeError):
            pass
    return res
```

**Propósito:** Lee la fecha guardada en `ir.config_parameter` y la convierte a tipo Date para mostrarla en el formulario.

**¿Por qué es necesario?** Porque `res.config.settings` es un modelo temporal. Los campos no se guardan automáticamente en una tabla; se deben guardar/leer manualmente de `ir.config_parameter`.

##### `set_values(self)`

```python
def set_values(self):
    super().set_values()
    value = ''
    if self.microsoft_outlook_secret_expiration:
        value = fields.Date.to_string(self.microsoft_outlook_secret_expiration)
    self.env['ir.config_parameter'].sudo().set_param(
        'microsoft_outlook_secret_expiration', value
    )
```

**Propósito:** Guarda la fecha en `ir.config_parameter` cuando el usuario hace clic en "Guardar" en Settings.

---

### Clase 2: `OutlookSecretNotifier`

```python
class OutlookSecretNotifier(models.AbstractModel):
    _name = 'outlook.secret.notifier'
    _description = 'Microsoft Outlook Secret Notifier'
```

**Propósito:** Contiene toda la lógica de verificación y notificación.

**Tipo de Modelo:** `AbstractModel` (modelo abstracto, NO crea tabla en BD)

**¿Por qué AbstractModel?** Porque solo necesitamos métodos, no almacenar datos. Es el patrón estándar en Odoo para "servicios" o "utilidades".

#### Constantes

```python
NOTIFY_DAYS_BEFORE = 30
```

**Propósito:** Define cuántos días antes de la expiración se debe notificar. Valor fijo (no configurable por el usuario).

#### Método Principal: `_cron_check_outlook_tokens(self)`

```python
@api.model
def _cron_check_outlook_tokens(self):
    """Daily cron: check expiration date AND validate active Outlook server tokens."""
```

**Propósito:** Es el punto de entrada llamado por el cron job.

**Decorador `@api.model`:** Indica que el método opera a nivel de modelo, no de registro individual.

**Flujo interno:**

1. **Verificar límite diario:**

    ```python
    last_notif = Config.get_param('outlook_notifier_last_date', '')
    if last_notif == today_str:
        return  # Ya notificó hoy, salir
    ```

2. **Verificar fecha de expiración manual:**

    ```python
    exp_str = Config.get_param('microsoft_outlook_secret_expiration', '')
    if exp_str:
        days_left = (exp_date - today).days
        if days_left <= NOTIFY_DAYS_BEFORE:
            notifications.append(...)  # Añadir alerta
    ```

3. **Validar tokens de servidores:**

    ```python
    token_errors = self._check_outlook_servers()
    notifications.extend(token_errors)
    ```

4. **Enviar notificaciones si hay alertas:**
    ```python
    if notifications:
        self._send_notifications(notifications)
        Config.set_param('outlook_notifier_last_date', today_str)
    ```

#### Método: `_check_outlook_servers(self)`

```python
def _check_outlook_servers(self):
    """Try to validate tokens on all active Outlook servers."""
```

**Propósito:** Recorre todos los servidores Outlook configurados y valida sus tokens.

**¿Cómo valida?** Llama a `_generate_outlook_oauth2_string()` del módulo `microsoft_outlook`. Este método:

1. Verifica si el access token está vigente
2. Si no, intenta renovarlo usando el refresh token
3. Si la renovación falla (secret expirado), lanza una excepción

**Servidores verificados:**

| Modelo             | Filtro                                         | Campo de usuario |
| ------------------ | ---------------------------------------------- | ---------------- |
| `ir.mail_server`   | `smtp_authentication = 'outlook'`              | `smtp_user`      |
| `fetchmail.server` | `server_type = 'outlook'` AND `state = 'done'` | `user`           |

**Manejo de errores:**

```python
try:
    server._generate_outlook_oauth2_string(server.smtp_user)
except Exception as e:
    errors.append(_('❌ Servidor saliente "%s": %s') % (server.name, str(e)[:100]))
```

#### Método: `_send_notifications(self, messages)`

```python
def _send_notifications(self, messages):
    """Send notifications via channel and email."""
```

**Propósito:** Envía las alertas acumuladas por dos canales.

**Canal 1: Discuss (Admin Channel)**

```python
channel = self.env.ref('mail.channel_admin', raise_if_not_found=False)
if channel:
    channel.sudo().message_post(body=body, message_type='notification')
```

- `mail.channel_admin` es el canal "#general" para administradores
- `message_post()` publica un mensaje en el canal
- `sudo()` evita problemas de permisos

**Canal 2: Email**

```python
admin_users = self.env['res.users'].sudo().search([
    ('groups_id', 'in', self.env.ref('base.group_system').id),
    ('email', '!=', False),
])
```

- Busca usuarios con grupo `base.group_system` (administradores)
- Que tengan email configurado
- Envía un email individual a cada uno

**Formato del email:**

```python
self.env['mail.mail'].sudo().create({
    'subject': _('🔔 Alerta Microsoft Outlook'),
    'body_html': f'<p>{body}</p>',
    'email_to': user.email,
    'email_from': email_from,
    'auto_delete': True,  # Se borra después de enviar
}).send()
```

---

## Configuración del Sistema

### System Parameters Utilizados

El módulo usa `ir.config_parameter` para almacenar configuración persistente:

| Parámetro                             | Propósito                                | Ejemplo      |
| ------------------------------------- | ---------------------------------------- | ------------ |
| `microsoft_outlook_secret_expiration` | Fecha de expiración del secret           | `2025-06-15` |
| `outlook_notifier_last_date`          | Última fecha de notificación (anti-spam) | `2025-12-05` |

**Ubicación en Odoo:** Settings → Technical → System Parameters

### ¿Cómo Modificarlos Manualmente?

1. Activar modo desarrollador
2. Ir a Settings → Technical → System Parameters
3. Buscar el parámetro
4. Modificar el valor

**Caso de uso:** Para forzar una nueva notificación (testing), eliminar o modificar `outlook_notifier_last_date`.

---

## Dependencias

### Dependencias Directas

```python
"depends": ["microsoft_outlook"]
```

| Módulo              | Propósito                      | ¿Por qué es necesario?                                        |
| ------------------- | ------------------------------ | ------------------------------------------------------------- |
| `microsoft_outlook` | Integración OAuth2 con Outlook | Proporciona `MicrosoftOutlookMixin` y la configuración OAuth2 |

### Dependencias Indirectas (ya incluidas por `microsoft_outlook`)

| Módulo | Propósito                                       |
| ------ | ----------------------------------------------- |
| `mail` | Sistema de mensajería de Odoo (canales, emails) |
| `base` | Funcionalidad base de Odoo (usuarios, permisos) |

### Modelos Externos Utilizados

| Modelo                | Módulo Origen                     | Uso en este módulo                      |
| --------------------- | --------------------------------- | --------------------------------------- |
| `ir.mail_server`      | `base` + `microsoft_outlook`      | Verificar servidores de correo saliente |
| `fetchmail.server`    | `fetchmail` + `microsoft_outlook` | Verificar servidores de correo entrante |
| `res.config.settings` | `base`                            | Heredado para añadir campo de fecha     |
| `ir.config_parameter` | `base`                            | Almacenar configuración                 |
| `mail.mail`           | `mail`                            | Enviar emails                           |
| `discuss.channel`     | `mail`                            | Publicar en canales                     |
| `res.users`           | `base`                            | Buscar administradores                  |

---

## Flujo de Ejecución

### Ejecución Automática (Diaria)

```
00:00 → Odoo Scheduler ejecuta cron jobs pendientes
         │
         ▼
      ir.cron ejecuta: model._cron_check_outlook_tokens()
         │
         ▼
      [Verificaciones...]
         │
         ▼
      Si hay alertas → Envía notificaciones
         │
         ▼
      Guarda fecha en outlook_notifier_last_date
```

### Ejecución Manual

1. Settings → Technical → Scheduled Actions
2. Buscar "Check Microsoft Outlook Tokens"
3. Click en "Run Manually"

**Nota:** Si ya notificó hoy, no volverá a notificar. Para forzar:

- Eliminar `outlook_notifier_last_date` de System Parameters

---

## Código Core vs No-Core

### CORE (Esencial - No eliminar)

| Elemento                            | Archivo               | Razón                     |
| ----------------------------------- | --------------------- | ------------------------- |
| Clase `OutlookSecretNotifier`       | `outlook_notifier.py` | Contiene toda la lógica   |
| Método `_cron_check_outlook_tokens` | `outlook_notifier.py` | Punto de entrada del cron |
| Método `_check_outlook_servers`     | `outlook_notifier.py` | Validación de tokens      |
| Método `_send_notifications`        | `outlook_notifier.py` | Envío de alertas          |
| Cron job                            | `ir_cron.xml`         | Dispara la verificación   |
| Security rules                      | `ir.model.access.csv` | Sin esto, el módulo falla |

### NO-CORE (Opcional - Se puede simplificar)

| Elemento                       | Archivo                         | Se puede eliminar si...              |
| ------------------------------ | ------------------------------- | ------------------------------------ |
| Clase `ResConfigSettings`      | `outlook_notifier.py`           | No necesitas configurar fecha manual |
| Vista de settings              | `res_config_settings_views.xml` | No necesitas UI para la fecha        |
| Constante `NOTIFY_DAYS_BEFORE` | `outlook_notifier.py`           | Si no usas fecha manual              |

### Simplificación Máxima

Si solo quieres detección automática de errores (sin fecha manual):

```python
# outlook_notifier.py simplificado
class OutlookSecretNotifier(models.AbstractModel):
    _name = 'outlook.secret.notifier'

    @api.model
    def _cron_check_outlook_tokens(self):
        Config = self.env['ir.config_parameter'].sudo()
        today_str = date.today().strftime('%Y-%m-%d')

        if Config.get_param('outlook_notifier_last_date', '') == today_str:
            return

        errors = self._check_outlook_servers()
        if errors:
            self._send_notifications(errors)
            Config.set_param('outlook_notifier_last_date', today_str)

    # ... resto de métodos
```

---

## Instalación

### Pre-requisitos

1. Odoo 17 o 18 instalado
2. Módulo `microsoft_outlook` instalado
3. Al menos un servidor Outlook configurado

### Pasos

1. **Copiar el módulo:**

    ```
    Copiar carpeta berpia_microsoft_outlook_notifier/ a:
    - addons/ (carpeta de addons personalizados)
    - O cualquier carpeta en addons_path
    ```

2. **Actualizar lista de apps:**
    - Ir a Apps
    - Menú ⋮ → Update Apps List

3. **Instalar:**
    - Buscar "Microsoft Outlook Token Notifier"
    - Click en Install

### Verificar Instalación

1. Ir a Settings → Technical → Scheduled Actions
2. Debe aparecer "Check Microsoft Outlook Tokens"
3. Ir a Settings → General Settings → Outlook
4. Debe aparecer campo "Secret Expires On"

---

## Configuración de Uso

### Configuración Mínima (Solo detección automática)

**No se requiere configuración.** El módulo detectará automáticamente cuando los tokens fallen.

### Configuración Recomendada (Con preaviso)

1. Ir a Azure Portal → App registrations → Tu App → Certificates & secrets
2. Anotar la fecha de expiración del client secret
3. En Odoo: Settings → General Settings → Outlook
4. Establecer "Secret Expires On" con esa fecha
5. Guardar

**Resultado:** Recibirás alertas 30 días antes de que expire.

---

## Testing Manual

### Test 1: Verificar que el cron funciona

1. Settings → Technical → Scheduled Actions
2. Buscar "Check Microsoft Outlook Tokens"
3. Click en "Run Manually"
4. Revisar logs del servidor

### Test 2: Forzar notificación

1. Settings → Technical → System Parameters
2. Buscar `outlook_notifier_last_date`
3. Eliminar o cambiar a fecha anterior
4. Ejecutar cron manualmente
5. Verificar:
    - Email recibido
    - Mensaje en Discuss → #general

### Test 3: Simular secret expirado

1. En Azure Portal, revocar o eliminar el client secret
2. Ejecutar cron manualmente
3. Debe generar notificación

---

## Troubleshooting

### Problema: No genera notificaciones

**Causa 1:** Ya notificó hoy

- **Solución:** Eliminar `outlook_notifier_last_date` de System Parameters

**Causa 2:** No hay servidores Outlook configurados

- **Solución:** Verificar que existe al menos un `ir.mail_server` con `smtp_authentication='outlook'` o `fetchmail.server` con `server_type='outlook'`

**Causa 3:** Los servidores no tienen refresh token

- **Solución:** Volver a autorizar los servidores en Settings → Outlook

### Problema: Email dice "noreply@localhost"

**Causa:** No hay email configurado en la compañía

- **Solución:** Settings → Companies → [Tu compañía] → Email

### Problema: No llega email pero sí Discuss

**Causa:** Problemas con el servidor de correo saliente

- **Solución:** Verificar configuración de outgoing mail server

### Ver logs del módulo

En el archivo de log de Odoo, buscar:

```
grep "Outlook" odoo.log
```

Mensajes típicos:

```
INFO: Outlook notification sent with 2 alerts.
DEBUG: Outlook notification already sent today.
ERROR: Failed to send email to admin@example.com: ...
```

---

## Mantenimiento y Actualizaciones

### Compatibilidad con Futuras Versiones de Odoo

El módulo está diseñado para ser **resistente a actualizaciones**:

| Aspecto                      | Estrategia                                                             |
| ---------------------------- | ---------------------------------------------------------------------- |
| Sin herencia de métodos core | No hace override de métodos del módulo `microsoft_outlook`             |
| Mínimas dependencias         | Solo depende de `microsoft_outlook`                                    |
| Modelos abstractos           | No crea tablas nuevas                                                  |
| API estable                  | Usa `ir.config_parameter`, `mail.mail`, `message_post` (APIs estables) |

### Si hay actualizaciones de `microsoft_outlook`

Verificar que sigan existiendo:

- `ir.mail_server.smtp_authentication` con valor `'outlook'`
- `fetchmail.server.server_type` con valor `'outlook'`
- Método `_generate_outlook_oauth2_string()` en los servidores

### Actualizar el módulo

```bash
./odoo-bin -u berpia_microsoft_outlook_notifier -d tu_base_de_datos
```

---

## Licencia

LGPL-3 (igual que Odoo Community)

---

## Autor

Pedro

---
