# Setup de Airtable para Validación NFTicket

> Configuración completa de la base de datos Airtable para automatizar el proceso de validación sin escribir código.

## 🗄️ Estructura de la Base

### Base Name: `NFTicket Validation`

## 📋 Tabla 1: Events

### Configuración de Campos

| Campo | Tipo | Configuración | Propósito |
|-------|------|---------------|-----------|
| **Event ID** | Auto number | Formato: EVT-{000} | ID único para tracking |
| **Event Name** | Single line text | Requerido | Nombre del evento |
| **Organizer Name** | Single line text | Requerido | Contacto principal |
| **Organizer Email** | Email | Requerido | Para comunicación |
| **Event Type** | Single select | Concert, Conference, Workshop, Party, Sports, Other | Categorización |
| **Event Date** | Date with time | Requerido | Cuándo es el evento |
| **Venue** | Single line text | Requerido | Dónde es el evento |
| **Expected Attendance** | Number | 1-500 | Tamaño del evento |
| **Demo Scheduled** | Date with time | | Cuándo es la demo |
| **Demo Recording** | URL | | Link a Zoom recording |
| **Status** | Single select | New Lead, Demo Scheduled, Demo Complete, Event Planned, Event Live, Event Complete, Lost | Pipeline tracking |

## 🔗 Configuración de Automatizaciones

### Automatización 1: Nueva Demo Programada

**Trigger**: Cuando "Demo Scheduled" se llena
**Acciones**:
1. Enviar email de confirmación al organizador
2. Crear evento en calendario
3. Agregar a Discord "NFTicket Pioneers"
4. Notificar al equipo en Slack

### Automatización 2: Demo Completada

**Trigger**: Cuando "Demo Recording" se agrega
**Acciones**:
1. Crear registro en tabla "Demo Feedback"
2. Enviar survey de feedback
3. Programar follow-up email en 48h
4. Actualizar métricas en Dashboard

## 📱 Configuración Mobile-First

### Formularios Optimizados

**Demo Request Form** (Airtable Form):
```
Campos visibles:
- Event Name (required)
- Organizer Name (required)  
- Email (required)
- Phone (optional)
- Event Type (dropdown)
- Expected Attendance (slider 10-500)
- Current Solution (dropdown)
- Preferred Demo Time (calendar picker)

Configuración:
- Mobile-responsive: ON
- Progress bar: ON
- Thank you message: "¡Gracias! Te contactaremos en 24h para confirmar tu demo personalizada."
```

---

**🎯 Resultado Esperado**: Sistema completo que automatiza 80% del proceso de validación, permitiendo enfoque total en demos y feedback de calidad.