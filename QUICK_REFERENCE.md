# Quick Reference: Email API Implementation

## 🔍 Code Overview

### Helper Functions Added to `app/main.py`

```python
import httpx
import os

def _send_via_resend(to_email: str, subject: str, text: str, html: str = None) -> bool:
    """Send email via Resend HTTP API (works on Render Free tier)"""
    api_key = os.getenv("RESEND_API_KEY")
    from_email = os.getenv("FROM_EMAIL", "noreply@skybook.com")
    
    if not api_key:
        return False
    
    try:
        payload = {
            "from": from_email,
            "to": [to_email],
            "subject": subject,
            "text": text,
        }
        
        if html:
            payload["html"] = html
        
        resp = httpx.post(
            "https://api.resend.com/emails",
            headers={"Authorization": f"Bearer {api_key}"},
            json=payload,
            timeout=15,
        )
        
        if resp.status_code in (200, 201):
            print(f"✅ Email sent via Resend to: {to_email}")
            return True
        else:
            print(f"⚠️ Resend API error: {resp.status_code} - {resp.text}")
            return False
            
    except Exception as e:
        print(f"⚠️ Resend error: {e}")
        return False


def _send_via_smtp(to_email: str, subject: str, body: str, booking: dict = None) -> bool:
    """Send email via Gmail SMTP (fallback for paid Render plans)"""
    try:
        smtp_server = os.getenv("SMTP_SERVER", "smtp.gmail.com")
        smtp_port = int(os.getenv("SMTP_PORT", 587))
        sender_email = os.getenv("SMTP_EMAIL")
        sender_password = os.getenv("SMTP_PASSWORD")

        if not sender_email or not sender_password:
            print("\n⚠️ SMTP credentials not configured in .env file")
            return False

        msg = MIMEMultipart()
        msg['From'] = sender_email
        msg['To'] = to_email
        msg['Subject'] = subject
        msg.attach(MIMEText(body, 'plain'))

        # Attach PDF and JSON if booking provided
        if booking:
            try:
                pdf_path = generate_booking_pdf(booking)
                with open(pdf_path, 'rb') as f:
                    pdf_attachment = MIMEApplication(f.read(), _subtype='pdf')
                    pdf_attachment.add_header('Content-Disposition', 'attachment', 
                                            filename=f'booking_{booking["confirmation_code"]}.pdf')
                    msg.attach(pdf_attachment)
            except Exception as pdf_err:
                print(f"⚠️ Error generating PDF: {pdf_err}")

            booking_json = {k: v for k, v in booking.items() if k not in ['payment']}
            json_attachment = MIMEApplication(
                json.dumps(booking_json, indent=2).encode('utf-8'),
                _subtype='json'
            )
            json_attachment.add_header('Content-Disposition', 'attachment', 
                                     filename=f'booking_{booking["confirmation_code"]}.json')
            msg.attach(json_attachment)

        try:
            print(f"\n📧 Attempting to send email via SMTP to {to_email}...")
            server = smtplib.SMTP(smtp_server, smtp_port, timeout=15)
            server.set_debuglevel(0)
            server.ehlo()
            server.starttls()
            server.ehlo()
            
            print(f"🔐 Logging in as {sender_email}...")
            server.login(sender_email, sender_password)
            
            print(f"📤 Sending email...")
            server.sendmail(sender_email, to_email, msg.as_string())
            server.quit()
            
            print(f"✅ EMAIL SUCCESSFULLY SENT VIA SMTP TO: {to_email}")
            return True
            
        except smtplib.SMTPAuthenticationError as auth_err:
            print("\n❌ SMTP Authentication Failed!")
            print(f"    Error: {auth_err}")
            print("\n💡 SOLUTION:")
            print("    For Gmail accounts:")
            print("    1. Enable 2-Factor Authentication on your Google account")
            print("    2. Generate an App Password at: https://myaccount.google.com/apppasswords")
            print("    3. Use the 16-character app password (no spaces) as SMTP_PASSWORD")
            print(f"    4. Current SMTP_EMAIL: {sender_email}")
            return False
            
        except Exception as send_err:
            print(f"\n❌ Error sending email via SMTP: {send_err}")
            return False
            
    except Exception as e:
        print(f"\n❌ SMTP configuration error: {e}")
        return False
```

### Updated Email Functions

```python
def send_confirmation_email(email: str, booking: dict) -> bool:
    """Send booking confirmation email (Resend first, SMTP fallback)"""
    try:
        subject = f"Flight Booking Confirmation - {booking['confirmation_code']}"
        
        body = f"""
Dear {booking['passenger']['first_name']} {booking['passenger']['last_name']},

Thank you for booking with SkyBook Airlines!

BOOKING CONFIRMATION
===================
Booking ID: {booking['booking_id']}
Confirmation Code: {booking['confirmation_code']}
Flight: {booking['flight_id']}

FLIGHT DETAILS
==============
Airline: {booking.get('airline', 'N/A')}
Route: {booking['origin']} ({booking.get('origin_city', '')}) → {booking['destination']} ({booking.get('destination_city', '')})
Departure: {booking['departure_time']}
Arrival: {booking['arrival_time']}
Duration: {booking['duration']}
Class: {booking['tier'].upper()}

PAYMENT INFORMATION
===================
Total Amount: ${booking['total_amount']:.2f}
Payment Status: Confirmed

IMPORTANT REMINDERS
==================
• Arrive at the airport 2 hours before departure
• Bring valid photo ID and confirmation code
• Check-in online 24 hours before departure

Have a great flight!

Best regards,
SkyBook Airlines Customer Service
        """

        # Try HTTP API first (works on Render Free tier)
        if _send_via_resend(email, subject, body):
            print(f"✅ EMAIL SENT TO: {email}")
            print(f"    Confirmation Code: {booking['confirmation_code']}")
            print(f"    Booking ID: {booking['booking_id']}")
            return True
        
        # Fallback to SMTP (only works on paid Render plans)
        if _send_via_smtp(email, subject, body, booking):
            print(f"✅ EMAIL SENT TO: {email}")
            print(f"    Confirmation Code: {booking['confirmation_code']}")
            print(f"    Booking ID: {booking['booking_id']}")
            return True
        
        # Neither method worked
        print(f"\n⚠️ Could not send confirmation email to {email}")
        print("   Please configure either:")
        print("   - RESEND_API_KEY and FROM_EMAIL for HTTP API (recommended for Render Free)")
        print("   - SMTP_EMAIL and SMTP_PASSWORD for SMTP (requires Render paid plan)")
        return False
            
    except Exception as e:
        print(f"\n❌ Email sending error: {e}")
        return False


def send_cancellation_email(email: str, booking: dict) -> bool:
    """Send booking cancellation email (Resend first, SMTP fallback)"""
    try:
        subject = f"Flight Booking Cancellation - {booking['booking_id']}"
        
        body = f"""
Dear {booking['passenger']['first_name']} {booking['passenger']['last_name']},

Your flight booking has been successfully cancelled.

CANCELLATION DETAILS
===================
Booking ID: {booking['booking_id']}
Flight: {booking['flight_id']}
Route: {booking['origin']} → {booking['destination']}
Departure: {booking['departure_time']}

REFUND INFORMATION
=================
Amount: ${booking['total_amount']:.2f}
Status: Processing
Expected Processing Time: 5-7 business days

Best regards,
SkyBook Airlines Customer Service
        """

        # Try HTTP API first (works on Render Free tier)
        if _send_via_resend(email, subject, body):
            print(f"✅ Cancellation email sent to: {email}")
            return True
        
        # Fallback to SMTP (only works on paid Render plans)
        if _send_via_smtp(email, subject, body):
            print(f"✅ Cancellation email sent to: {email}")
            return True
        
        # Neither method worked
        print(f"\n⚠️ Could not send cancellation email to {email}")
        print("   Please configure either:")
        print("   - RESEND_API_KEY and FROM_EMAIL for HTTP API (recommended for Render Free)")
        print("   - SMTP_EMAIL and SMTP_PASSWORD for SMTP (requires Render paid plan)")
        return False
            
    except Exception as e:
        print(f"\n❌ Cancellation email error: {e}")
        return False
```

## 🌍 Environment Variables

### For Resend (Recommended)
```env
RESEND_API_KEY=re_your_api_key_here
FROM_EMAIL=noreply@yourdomain.com
```

### For SMTP (Fallback)
```env
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_EMAIL=your.email@gmail.com
SMTP_PASSWORD=xxxx xxxx xxxx xxxx  # 16-char app password
```

## 🧪 Testing Locally

### 1. Create .env file:
```bash
cat > .env << EOF
RESEND_API_KEY=re_your_key_here
FROM_EMAIL=noreply@yourdomain.com
EOF
```

### 2. Run the app:
```bash
python -m uvicorn app.main:app --reload
```

### 3. Make a test booking:
- Visit http://localhost:8000
- Register new account
- Search flights
- Book a flight
- Check your email inbox for confirmation

## 📋 Files Modified

- `app/main.py` - Added helper functions and updated send_confirmation_email and send_cancellation_email

## 📋 Files Added

- `EMAIL_SETUP.md` - Comprehensive setup guide
- `RESEND_IMPLEMENTATION.md` - Implementation details
- `QUICK_REFERENCE.md` - This file

## ✅ Validation Checklist

- [x] Code compiles without errors
- [x] `httpx` is in requirements.txt (0.25.1)
- [x] No new external dependencies added
- [x] Handles missing environment variables gracefully
- [x] Provides clear error messages
- [x] Includes timeout protection (15 seconds)
- [x] Works on Render Free tier (Resend method)
- [x] Falls back to SMTP if Resend fails
- [x] Preserves PDF/JSON attachments (SMTP only)
- [x] Tested with confirmation and cancellation flows

## 🚀 Deployment Steps

1. **Get Resend API key** (free, 100 emails/day)
   - Sign up: https://resend.com/signup
   - Create key: https://resend.com/api-keys
   - Verify sender: Check Resend dashboard

2. **Update Render environment variables**
   - Go to: https://dashboard.render.com
   - Select your service
   - Environment tab
   - Add: `RESEND_API_KEY` and `FROM_EMAIL`

3. **Redeploy service**
   - Clear cache (optional)
   - Manual Deploy to latest commit

4. **Test**
   - Register and book a flight
   - Check email for confirmation

Done! 🎉
