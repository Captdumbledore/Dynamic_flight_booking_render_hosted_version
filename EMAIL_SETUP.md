# Email Configuration Guide

Your Flight Booking application now supports **two email sending methods**, with automatic fallback:

## 📧 Option A: Resend HTTP API (✅ Recommended for Render Free Tier)

**Resend** is a modern email API with a generous free tier. It works perfectly on Render's free plan.

### Setup Steps:

1. **Create a Resend account** (free): https://resend.com/signup
2. **Create an API key**:
   - Go to https://resend.com/api-keys
   - Click "Create API Key"
   - Copy the key

3. **Add environment variables to Render**:
   - Go to your Render service dashboard
   - Navigate to **Environment** section
   - Add these variables:
     ```
     RESEND_API_KEY=<your_api_key_here>
     FROM_EMAIL=noreply@yourdomain.com   # Use your verified Resend sender
     ```
   - Click "Save"

4. **Verify sender email in Resend** (if using custom domain):
   - Resend will email you a verification link
   - Click it to activate your sender address

### Cost:
- **Free tier**: 100 emails/day
- **Paid**: $0.20 per email (pay-as-you-go)

---

## 📧 Option B: Gmail SMTP (Fallback for Paid Render Plans)

SMTP uses port 25 for outbound email. This is **blocked on Render's free tier** but works on paid plans.

### Setup Steps (if you upgrade to paid Render):

1. **Enable 2-Factor Authentication on Gmail**:
   - Go to https://myaccount.google.com/security
   - Enable 2-Step Verification

2. **Generate an App Password**:
   - Go to https://myaccount.google.com/apppasswords
   - Select "Mail" and "Windows Computer"
   - Copy the 16-character password

3. **Add environment variables to Render**:
   ```
   SMTP_SERVER=smtp.gmail.com
   SMTP_PORT=587
   SMTP_EMAIL=your.email@gmail.com
   SMTP_PASSWORD=<16_char_app_password>
   ```

---

## 🔄 How the System Works

When a booking is confirmed or cancelled, the app:

1. **Tries Resend first** (HTTP API - works everywhere)
   - If `RESEND_API_KEY` is set → sends email via Resend
   - Instant & reliable

2. **Falls back to SMTP** if Resend fails
   - Only if `SMTP_EMAIL` and `SMTP_PASSWORD` are set
   - Provides redundancy

3. **Graceful degradation**
   - If neither is configured → logs the email content (development mode)
   - Shows clear instructions on what to configure

---

## 🧪 Testing

### Test Resend Locally:
```bash
# Set your test keys
export RESEND_API_KEY="re_xxxxx"
export FROM_EMAIL="test@yourdomain.com"

# Run the app - make a booking and email will be sent
python -m uvicorn app.main:app --reload
```

### Test SMTP Locally:
```bash
export SMTP_EMAIL="your.email@gmail.com"
export SMTP_PASSWORD="xxxx xxxx xxxx xxxx"

# Make a booking and watch the SMTP logs
```

---

## 🐛 Troubleshooting

### "Resend error" message:
- Check that `RESEND_API_KEY` is set correctly in Render
- Verify `FROM_EMAIL` is a verified sender in Resend dashboard
- Check Render logs for the specific error

### "SMTP Authentication Failed":
- Verify you're using a Gmail **App Password**, not your regular password
- App Password must have no spaces (use copy-paste)
- Only works on Render **paid plans** (SMTP port blocked on free tier)

### Emails not sending at all:
- Check Render logs: https://dashboard.render.com → select service → Logs
- Ensure at least one email method is configured
- For production, always set up Resend (it's free and reliable)

---

## 📊 Production Recommendation

| Scenario | Method | Why |
|----------|--------|-----|
| **Free tier** | ✅ Resend | Only option that works |
| **Paid tier** | ✅ Resend + SMTP | Redundancy, Resend is cheaper |
| **High volume** | 🔄 Both | Failover protection |
| **Development** | 📝 Log to console | Free, no setup needed |

---

## 💡 Best Practices

1. **Always use Resend on Render Free** - it's the only reliable option
2. **Add SMTP as backup only if upgrading to paid** - provides failover
3. **Monitor Render logs** - watch for email errors in logs
4. **Test before going live** - make a test booking before launch
5. **Set `FROM_EMAIL` to a domain you own** - improves deliverability

---

## 🔗 Useful Links

- **Resend Docs**: https://resend.com/docs
- **Resend Dashboard**: https://resend.com/emails
- **Render Dashboard**: https://dashboard.render.com
- **Gmail App Passwords**: https://myaccount.google.com/apppasswords
