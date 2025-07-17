# Setup de Supabase para Validación NFTicket

> Configuración completa de Supabase para automatizar el proceso de validación con una base de datos real y APIs desde día 1.

## 🗄️ Estructura de la Base de Datos

### Database: `nfticket_validation`

## 📋 Tabla 1: events

```sql
-- EVENTS TABLE
CREATE TABLE events (
  id BIGSERIAL PRIMARY KEY,
  event_name VARCHAR(200) NOT NULL,
  organizer_name VARCHAR(100) NOT NULL,
  organizer_email VARCHAR(150) NOT NULL,
  organizer_phone VARCHAR(20),
  event_type VARCHAR(50) CHECK (event_type IN ('concert', 'conference', 'workshop', 'party', 'sports', 'other')),
  event_date TIMESTAMPTZ,
  venue VARCHAR(200),
  expected_attendance INTEGER CHECK (expected_attendance > 0),
  current_solution VARCHAR(100),
  demo_scheduled TIMESTAMPTZ,
  demo_recording_url TEXT,
  status VARCHAR(20) DEFAULT 'new_lead' CHECK (status IN ('new_lead', 'demo_scheduled', 'demo_complete', 'event_planned', 'event_live', 'event_complete', 'lost')),
  source VARCHAR(50) DEFAULT 'landing_page',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS (Row Level Security)
ALTER TABLE events ENABLE ROW LEVEL SECURITY;

-- Create policy for public access (for demo form)
CREATE POLICY "Allow public insert" ON events FOR INSERT WITH CHECK (true);
CREATE POLICY "Allow authenticated read" ON events FOR SELECT USING (auth.role() = 'authenticated');
```

## 📊 Tabla 2: demo_feedback

```sql
-- DEMO FEEDBACK TABLE
CREATE TABLE demo_feedback (
  id BIGSERIAL PRIMARY KEY,
  event_id BIGINT REFERENCES events(id) ON DELETE CASCADE,
  demo_date TIMESTAMPTZ DEFAULT NOW(),
  demo_duration_minutes INTEGER,
  excitement_level INTEGER CHECK (excitement_level BETWEEN 1 AND 10),
  comprehension VARCHAR(30) CHECK (comprehension IN ('understood_immediately', 'needed_clarification', 'confused', 'very_confused')),
  setup_time_perception VARCHAR(20) CHECK (setup_time_perception IN ('under_5min', '5_10min', '10_15min', 'over_15min', 'too_complex')),
  biggest_concern VARCHAR(30) CHECK (biggest_concern IN ('price', 'tech_complexity', 'user_adoption', 'trust_security', 'integration', 'other')),
  concern_details TEXT,
  price_reaction VARCHAR(20) CHECK (price_reaction IN ('great_value', 'acceptable', 'expensive', 'too_expensive', 'need_more_info')),
  willingness_to_pay VARCHAR(20) CHECK (willingness_to_pay IN ('yes_2_5', 'yes_3', 'yes_4', 'only_if_lower', 'no')),
  feature_priority_1 VARCHAR(30),
  feature_priority_2 VARCHAR(30),
  feature_priority_3 VARCHAR(30),
  questions_asked TEXT,
  positive_reactions TEXT,
  next_steps VARCHAR(30) CHECK (next_steps IN ('schedule_real_event', 'join_discord', 'need_more_info', 'follow_up_later', 'not_interested')),
  referral_intent VARCHAR(20) CHECK (referral_intent IN ('will_definitely_refer', 'might_refer', 'probably_not', 'wont_refer')),
  referral_names TEXT,
  overall_score DECIMAL(3,1), -- Calculated field
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE demo_feedback ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Allow authenticated access" ON demo_feedback FOR ALL USING (auth.role() = 'authenticated');
```

## 🎯 Tabla 3: real_events

```sql
-- REAL EVENTS TABLE (cuando se ejecutan eventos reales)
CREATE TABLE real_events (
  id BIGSERIAL PRIMARY KEY,
  event_id BIGINT REFERENCES events(id) ON DELETE CASCADE,
  go_live_date TIMESTAMPTZ DEFAULT NOW(),
  tickets_available INTEGER NOT NULL DEFAULT 0,
  tickets_sold INTEGER DEFAULT 0,
  checkins_completed INTEGER DEFAULT 0,
  revenue_generated DECIMAL(10,2) DEFAULT 0,
  commission_earned DECIMAL(10,2) GENERATED ALWAYS AS (revenue_generated * 0.025) STORED,
  issues_reported TEXT,
  issue_severity VARCHAR(20) CHECK (issue_severity IN ('none', 'minor', 'major', 'critical')),
  resolution_time_minutes INTEGER,
  user_feedback TEXT,
  organizer_nps INTEGER CHECK (organizer_nps BETWEEN 1 AND 10),
  photo_evidence_urls TEXT[], -- Array of URLs
  success_rate DECIMAL(3,2) GENERATED ALWAYS AS (
    CASE 
      WHEN tickets_sold > 0 THEN ROUND((checkins_completed::decimal / tickets_sold), 2)
      ELSE 0 
    END
  ) STORED,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE real_events ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Allow authenticated access" ON real_events FOR ALL USING (auth.role() = 'authenticated');
```

## 📈 Views para Analytics

```sql
-- ANALYTICS VIEW
CREATE VIEW validation_analytics AS
SELECT 
  -- Demo metrics
  COUNT(DISTINCT e.id) as total_demos,
  AVG(df.excitement_level) as avg_excitement,
  COUNT(CASE WHEN df.next_steps = 'schedule_real_event' THEN 1 END)::decimal / COUNT(df.id) as conversion_rate,
  COUNT(CASE WHEN df.willingness_to_pay = 'yes_2_5' THEN 1 END)::decimal / COUNT(df.id) as price_acceptance_rate,
  
  -- Event metrics
  COUNT(DISTINCT re.id) as total_real_events,
  SUM(re.tickets_sold) as total_tickets_sold,
  SUM(re.revenue_generated) as total_revenue,
  SUM(re.commission_earned) as total_commission,
  AVG(re.success_rate) as avg_success_rate,
  AVG(re.organizer_nps) as avg_organizer_nps,
  
  -- Referral metrics
  COUNT(CASE WHEN df.referral_names IS NOT NULL AND df.referral_names != '' THEN 1 END) as referrals_provided
  
FROM events e
LEFT JOIN demo_feedback df ON e.id = df.event_id
LEFT JOIN real_events re ON e.id = re.event_id;
```

## 🔗 Supabase Edge Functions para Automatización

### Function 1: Demo Request Handler

```typescript
// supabase/functions/demo-request/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  if (req.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 })
  }

  const { 
    eventName, 
    organizerName, 
    organizerEmail, 
    phone, 
    eventType, 
    expectedAttendance,
    currentSolution,
    preferredDemoTime 
  } = await req.json()

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
  )

  // Insert event
  const { data: event, error } = await supabase
    .from('events')
    .insert({
      event_name: eventName,
      organizer_name: organizerName,
      organizer_email: organizerEmail,
      organizer_phone: phone,
      event_type: eventType,
      expected_attendance: expectedAttendance,
      current_solution: currentSolution,
      demo_scheduled: preferredDemoTime,
      status: 'demo_scheduled'
    })
    .select()
    .single()

  if (error) {
    return new Response(JSON.stringify({ error: error.message }), { 
      status: 400,
      headers: { 'Content-Type': 'application/json' }
    })
  }

  // Send confirmation email (integrate with Resend/SendGrid)
  await sendConfirmationEmail(organizerEmail, organizerName, preferredDemoTime)
  
  // Create Discord invite
  await createDiscordInvite(organizerEmail, organizerName)

  return new Response(JSON.stringify({ 
    success: true, 
    eventId: event.id,
    message: 'Demo programada exitosamente' 
  }), {
    headers: { 'Content-Type': 'application/json' }
  })
})

async function sendConfirmationEmail(email: string, name: string, demoTime: string) {
  // Implementation with email service
}

async function createDiscordInvite(email: string, name: string) {
  // Implementation with Discord API
}
```

### Function 2: Demo Feedback Processor

```typescript
// supabase/functions/demo-feedback/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  const feedback = await req.json()
  
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
  )

  // Calculate overall score
  const overallScore = calculateOverallScore(feedback)
  
  // Insert feedback
  const { error } = await supabase
    .from('demo_feedback')
    .insert({
      ...feedback,
      overall_score: overallScore
    })

  // Update event status
  await supabase
    .from('events')
    .update({ status: 'demo_complete' })
    .eq('id', feedback.event_id)

  // Trigger follow-up actions based on score
  if (overallScore >= 8) {
    await scheduleFollowUp(feedback.event_id, 'high_priority')
  }

  return new Response(JSON.stringify({ success: true }))
})

function calculateOverallScore(feedback: any) {
  const weights = {
    excitement_level: 0.3,
    willingness_to_pay: 0.4,
    next_steps: 0.3
  }
  
  let payScore = 0
  switch(feedback.willingness_to_pay) {
    case 'yes_2_5': payScore = 10; break
    case 'yes_3': payScore = 8; break
    case 'yes_4': payScore = 6; break
    case 'only_if_lower': payScore = 4; break
    default: payScore = 0
  }
  
  let stepScore = 0
  switch(feedback.next_steps) {
    case 'schedule_real_event': stepScore = 10; break
    case 'join_discord': stepScore = 8; break
    case 'need_more_info': stepScore = 6; break
    default: stepScore = 2
  }
  
  return (feedback.excitement_level * weights.excitement_level) +
         (payScore * weights.willingness_to_pay) +
         (stepScore * weights.next_steps)
}
```

## 🎨 Frontend Integration con Supabase

### Demo Request Form (React)

```typescript
// components/DemoRequestForm.tsx
import { useState } from 'react'
import { supabase } from '@/lib/supabase'

export function DemoRequestForm() {
  const [formData, setFormData] = useState({
    eventName: '',
    organizerName: '',
    organizerEmail: '',
    phone: '',
    eventType: '',
    expectedAttendance: '',
    currentSolution: '',
    preferredDemoTime: ''
  })
  
  const [loading, setLoading] = useState(false)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)

    try {
      const { data, error } = await supabase.functions.invoke('demo-request', {
        body: formData
      })

      if (error) throw error

      // Show success message
      alert('¡Demo programada! Te contactaremos en 24h.')
      
      // Redirect to thank you page
      window.location.href = '/gracias'
      
    } catch (error) {
      console.error('Error:', error)
      alert('Hubo un error. Por favor intenta de nuevo.')
    } finally {
      setLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input
        type="text"
        placeholder="Nombre del evento"
        value={formData.eventName}
        onChange={(e) => setFormData({...formData, eventName: e.target.value})}
        className="w-full p-3 border rounded-lg"
        required
      />
      
      <input
        type="text"
        placeholder="Tu nombre"
        value={formData.organizerName}
        onChange={(e) => setFormData({...formData, organizerName: e.target.value})}
        className="w-full p-3 border rounded-lg"
        required
      />
      
      <input
        type="email"
        placeholder="Email"
        value={formData.organizerEmail}
        onChange={(e) => setFormData({...formData, organizerEmail: e.target.value})}
        className="w-full p-3 border rounded-lg"
        required
      />
      
      <select
        value={formData.eventType}
        onChange={(e) => setFormData({...formData, eventType: e.target.value})}
        className="w-full p-3 border rounded-lg"
        required
      >
        <option value="">Tipo de evento</option>
        <option value="concert">Concierto</option>
        <option value="conference">Conferencia</option>
        <option value="workshop">Workshop</option>
        <option value="party">Fiesta</option>
        <option value="sports">Deportes</option>
        <option value="other">Otro</option>
      </select>
      
      <input
        type="number"
        placeholder="Asistentes esperados"
        value={formData.expectedAttendance}
        onChange={(e) => setFormData({...formData, expectedAttendance: e.target.value})}
        className="w-full p-3 border rounded-lg"
        min="1"
        required
      />
      
      <button
        type="submit"
        disabled={loading}
        className="w-full bg-green-500 text-white p-3 rounded-lg hover:bg-green-600 disabled:opacity-50"
      >
        {loading ? 'Programando...' : 'Programar Demo Gratis'}
      </button>
    </form>
  )
}
```

## 📊 Real-time Analytics Dashboard

```typescript
// components/ValidationDashboard.tsx
import { useEffect, useState } from 'react'
import { supabase } from '@/lib/supabase'

export function ValidationDashboard() {
  const [analytics, setAnalytics] = useState(null)
  const [events, setEvents] = useState([])

  useEffect(() => {
    fetchAnalytics()
    fetchRecentEvents()

    // Subscribe to real-time updates
    const channel = supabase
      .channel('validation_updates')
      .on('postgres_changes', 
        { event: '*', schema: 'public', table: 'events' },
        () => fetchAnalytics()
      )
      .subscribe()

    return () => {
      supabase.removeChannel(channel)
    }
  }, [])

  const fetchAnalytics = async () => {
    const { data } = await supabase
      .from('validation_analytics')
      .select('*')
      .single()
    
    setAnalytics(data)
  }

  const fetchRecentEvents = async () => {
    const { data } = await supabase
      .from('events')
      .select(`
        *,
        demo_feedback(excitement_level, overall_score),
        real_events(tickets_sold, revenue_generated)
      `)
      .order('created_at', { ascending: false })
      .limit(10)
    
    setEvents(data)
  }

  if (!analytics) return <div>Loading...</div>

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-semibold">Demos Realizadas</h3>
        <p className="text-3xl font-bold text-green-600">{analytics.total_demos}</p>
      </div>
      
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-semibold">Excitement Promedio</h3>
        <p className="text-3xl font-bold text-blue-600">
          {analytics.avg_excitement?.toFixed(1)}/10
        </p>
      </div>
      
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-semibold">Tasa de Conversión</h3>
        <p className="text-3xl font-bold text-purple-600">
          {(analytics.conversion_rate * 100)?.toFixed(1)}%
        </p>
      </div>
      
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-semibold">Aceptación Precio 2.5%</h3>
        <p className="text-3xl font-bold text-green-600">
          {(analytics.price_acceptance_rate * 100)?.toFixed(1)}%
        </p>
      </div>
      
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-semibold">Revenue Total</h3>
        <p className="text-3xl font-bold text-green-600">
          ${analytics.total_revenue?.toFixed(2)}
        </p>
      </div>
      
      <div className="bg-white p-6 rounded-lg shadow">
        <h3 className="text-lg font-semibold">NPS Promedio</h3>
        <p className="text-3xl font-bold text-blue-600">
          {analytics.avg_organizer_nps?.toFixed(1)}
        </p>
      </div>
    </div>
  )
}
```

## 🚀 Ventajas de Supabase vs Airtable

### ✅ **Supabase Advantages**
- **Real-time updates** via WebSockets
- **Serverless functions** para automatización
- **PostgreSQL completo** con queries complejas
- **APIs automáticas** REST y GraphQL
- **Escalabilidad** real para producción
- **Row Level Security** nativo
- **Integración directa** con frontend
- **No límites** de records o requests

### ❌ **Airtable Limitations Avoided**
- Limited API rate limits
- No real-time updates
- Expensive scaling costs
- No custom logic/functions
- Limited query capabilities
- Not production-ready for scaling

---

**🎯 Resultado**: Base de datos profesional lista para escalar desde validación hasta millones de usuarios, con automatización completa y analytics en tiempo real.