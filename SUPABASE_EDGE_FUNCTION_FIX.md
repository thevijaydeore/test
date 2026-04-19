# Supabase Edge Function CORS Fix

## Issue

The `updateLatLng` Edge Function is blocking requests from `localhost` due to missing CORS headers:

```
Access to fetch at 'https://fcitzuumiszlsrxfomlr.supabase.co/functions/v1/updateLatLng' 
from origin 'http://localhost:8081' has been blocked by CORS policy
```

## Solution

Update your `updateLatLng` Supabase Edge Function to include CORS headers.

### Option 1: Quick Fix (Add CORS Headers to Existing Function)

If you have access to the Edge Function code, add these headers to the response:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'

serve(async (req) => {
  // Handle CORS preflight requests
  if (req.method === 'OPTIONS') {
    return new Response('ok', {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'POST, OPTIONS',
        'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
      },
    })
  }

  try {
    // Your existing function logic here
    const { booking_id, lat, lng } = await req.json()
    
    // ... your database update logic ...

    return new Response(
      JSON.stringify({ success: true }),
      {
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*', // Add this
          'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type', // Add this
        },
      },
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      {
        status: 400,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*', // Add this
          'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type', // Add this
        },
      },
    )
  }
})
```

### Option 2: Complete Example Function

Here's a complete example of the `updateLatLng` function with proper CORS:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

serve(async (req) => {
  // Handle CORS preflight
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    const { booking_id, lat, lng } = await req.json()

    // Create Supabase client
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    )

    // Update the booking with lat/lng
    const { data, error } = await supabaseClient
      .from('your_bookings_table') // Replace with your table name
      .update({ latitude: lat, longitude: lng })
      .eq('id', booking_id)

    if (error) throw error

    return new Response(
      JSON.stringify({ success: true, data }),
      {
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
        status: 200,
      },
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      {
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
        status: 400,
      },
    )
  }
})
```

## Deployment Steps

1. Navigate to your Supabase project
2. Go to **Edge Functions** in the dashboard
3. Find the `updateLatLng` function
4. Update the code with CORS headers
5. Deploy the function
6. Test again from your frontend

## Alternative: Backend Proxy

If you can't modify the Edge Function, you could create a backend proxy that adds CORS headers, but this defeats the purpose of having a frontend-only solution.

## Testing After Fix

After updating the Edge Function:

1. Clear browser cache
2. Restart your dev server (`npm run dev`)
3. Submit the booking form again
4. Check the Network tab - the `updateLatLng` request should now succeed with status 200

## Current Workaround

The frontend code has been updated to handle this error gracefully:
- ✅ Booking is still created successfully
- ✅ User sees success message
- ⚠️ Location update fails silently (logged to console)
- The booking exists in your database, but might need manual location entry

