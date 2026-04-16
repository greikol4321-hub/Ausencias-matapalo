# INSTRUCCIONES PARA CONFIGURAR LA PLANTILLA EN EMAILJS

## Paso 1: Acceder a EmailJS
1. Ve a https://www.emailjs.com/
2. Inicia sesión con tu cuenta

## Paso 2: Crear/Editar el Template
1. En el panel de EmailJS, ve a **Email Templates**
2. Si ya tienes el template `template_fbik8ht`, haz clic en **Edit**
3. Si no, crea uno nuevo con **Create New Template**

## Paso 3: Configurar el Template

### A) NOMBRE DEL TEMPLATE
- Template ID: `template_htbp6yd`
- Template Name: `Notificación de Ausencia Docente`

### B) CONFIGURACIÓN DEL "TO EMAIL"
En el campo **To Email**, pon el correo al que quieres que lleguen las notificaciones:
```
direccion@ctpmatapalo.ed.cr
```
(O el correo que uses para recibir notificaciones)

### C) SUBJECT (ASUNTO DEL EMAIL)
```
🔔 Nueva Ausencia Registrada - {{nombre}}
```

### D) CONTENT (CUERPO DEL EMAIL)
1. Haz clic en el editor de contenido
2. Cambia a modo **HTML** (hay un botón que dice "Source" o "</>" en el editor)
3. **BORRA TODO** el contenido que hay
4. **COPIA Y PEGA** todo el contenido del archivo `plantilla-emailjs.html`

## Paso 4: Verificar las Variables
Asegúrate de que estas variables estén configuradas en la sección de variables del template:

- `{{nombre}}` - Nombre del docente
- `{{fecha_inicio}}` - Fecha de inicio de la ausencia
- `{{fecha_fin}}` - Fecha de finalización
- `{{motivo}}` - Motivo de la ausencia
- `{{detalle}}` - Detalle o justificación
- `{{actividad}}` - Actividad académica de remplazo

## Paso 5: Configurar el Service
1. Ve a **Email Services**
2. Verifica que tengas el service `service_lww6198` conectado
3. Si no, conéctalo a tu cuenta de correo (Gmail, Outlook, etc.)

## Paso 6: Probar el Template
1. Haz clic en **Test It** en EmailJS
2. Llena los valores de prueba:
   - nombre: Juan Pérez
   - fecha_inicio: 2026-04-17
   - fecha_fin: 2026-04-23
   - motivo: Cita Médica
   - detalle: Cita médica programada
   - actividad: Trabajar en el último cotidiano de clase
3. Haz clic en **Send Test Email**
4. Verifica que el correo llegue correctamente

## Paso 7: Guardar
1. Haz clic en **Save** para guardar los cambios
2. El template estará listo para usarse

## CONFIGURACIÓN ACTUAL EN EL CÓDIGO

El código en `docentes.html` ya está configurado con:

```javascript
emailjs.send("service_lww6198", "template_htbp6yd", {
    nombre: datos.nombre,
    fecha_inicio: datos.fecha,
    fecha_fin: datos.fecha_fin,
    motivo: datos.razon,
    detalle: datos.detalle,
    actividad: datos.actividad
})
```

## NOTAS IMPORTANTES

- **Public Key**: Ya está configurada en el código: `qAglmSdD5me55ae1-`
- **Service ID**: `service_2wecd52` (verificar que esté activo en EmailJS)
- **Template ID**: `template_fbik8ht` (usar este mismo ID al crear/editar)

## SOLUCIÓN DE PROBLEMAS

Si los correos no llegan:
1. Verifica que el Service esté conectado y activo
2. Revisa la cuota de correos de EmailJS (plan gratuito: 200 emails/mes)
3. Verifica la consola del navegador para ver errores
4. Revisa la bandeja de SPAM del correo destino

---

**¡Listo!** Una vez configurada la plantilla en EmailJS, cada vez que un docente registre una ausencia en el sistema, se enviará automáticamente este correo profesional a dirección.
