# Render Deployment Guide for Flight Booking Application

## Step-by-Step Instructions

### Step 1: Create a Render Account
1. Go to https://render.com
2. Click "Get Started" or "Sign Up"
3. Sign up using GitHub (recommended) or email
4. Verify your email

### Step 2: Connect Your GitHub Repository
1. Log in to Render dashboard
2. Click on "Dashboard" (top left)
3. Click "New +" button
4. Select "Web Service"
5. Click "Connect account" to link your GitHub
6. Authorize Render to access your GitHub repositories
7. Search for and select: `Dynamic_flight_booking_render_hosted_version`
8. Click "Connect"

### Step 3: Configure Web Service Settings
Fill in the following details:

**Basic Settings:**
- **Name:** `flight-booking-api` (or any name you prefer)
- **Environment:** `Python 3.11` (or latest Python 3)
- **Region:** Choose closest to your users (e.g., `Oregon (US West)`)
- **Branch:** `main`

**Build Command:**
```
pip install -r requirements.txt
```

**Start Command:**
```
gunicorn -c gunicorn_conf.py app.main:app
```
(or if you want to use Uvicorn directly:)
```
uvicorn app.main:app --host 0.0.0.0 --port $PORT
```

### Step 4: Add Environment Variables
1. Scroll down to "Environment Variables" section
2. Click "Add Environment Variable"
3. Add the following variables (check your `.env` file for values):

```
DATABASE_URL=your_database_url_here
AMADEUS_CLIENT_ID=your_amadeus_client_id
AMADEUS_CLIENT_SECRET=your_amadeus_client_secret
SECRET_KEY=your_secret_key_here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
MAIL_USERNAME=your_email@gmail.com
MAIL_PASSWORD=your_app_specific_password
MAIL_FROM=your_email@gmail.com
MAIL_PORT=587
MAIL_SERVER=smtp.gmail.com
```

### Step 5: Configure Build and Deployment
1. Scroll to "Build & Deploy" section
2. **Auto-Deploy:** Toggle ON (to auto-deploy when you push to main)
3. **Build Filter:** Leave empty
4. **Root Directory:** Leave as `/` (root)

### Step 6: Modify render.yaml (Optional but Recommended)
Create/update `render.yaml` file with:

```yaml
services:
  - type: web
    name: flight-booking-api
    env: python
    plan: free
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn app.main:app --host 0.0.0.0 --port $PORT
    envVars:
      - key: DATABASE_URL
        scope: build
      - key: AMADEUS_CLIENT_ID
        scope: build
      - key: AMADEUS_CLIENT_SECRET
        scope: build
      - key: SECRET_KEY
        scope: build
```

### Step 7: Set Up Database (If Using External Database)
**Option A: Using PostgreSQL on Render**
1. From Dashboard, click "New +"
2. Select "PostgreSQL"
3. Fill in name: `flight-booking-db`
4. Keep default settings
5. Create
6. Copy the database URL
7. Go back to your web service
8. Add `DATABASE_URL` environment variable with the copied URL

**Option B: Using SQLite (Already Configured)**
- Your app uses SQLite by default, no additional setup needed

### Step 8: Review and Deploy
1. Scroll to bottom and click "Create Web Service"
2. Render will start building your application
3. Watch the build logs for any errors
4. Once deployment is complete, you'll get a URL like: `https://flight-booking-api.onrender.com`

### Step 9: Test Your Deployment
1. Once deployed, visit: `https://your-service-name.onrender.com/docs`
2. This should show the FastAPI Swagger documentation
3. Test the endpoints to ensure everything works

### Step 10: Frontend Configuration (If Needed)
If you want to serve your frontend on Render:
1. Update CORS settings in `app/main.py` to allow your Render domain
2. Or deploy frontend separately on Render/Netlify/Vercel

## Important Notes

### Database Persistence
- **SQLite:** Data is stored locally, will be lost when Render restarts
- **PostgreSQL:** Data persists across restarts (recommended for production)

### Memory and Performance
- **Free Tier:** 0.5GB RAM, auto-spins down after 15 mins of inactivity
- **Paid Plans:** Better performance and always-on availability

### Cost
- Free tier is limited but good for testing
- Check Render pricing for production use

### Common Issues

**Issue: Application crashes on startup**
- Check logs in Render dashboard
- Verify all environment variables are set
- Ensure requirements.txt has all dependencies

**Issue: Database connection errors**
- Verify DATABASE_URL is correct
- Check if database service is running
- Ensure credentials are correct

**Issue: Timeout errors**
- Increase timeout in Render dashboard
- Check if your API calls are taking too long
- Optimize database queries

### Updating Your Application
1. Make changes to your code locally
2. Push to GitHub:
   ```
   git add .
   git commit -m "Your message"
   git push origin main
   ```
3. Render automatically redeploys (if auto-deploy is enabled)
4. Check deployment logs in Render dashboard

### Monitoring
- Visit Render dashboard
- Click on your service
- View "Logs" to see application output
- View "Metrics" for CPU, memory, and network usage

## Next Steps
1. Complete all deployment steps above
2. Test your API endpoints
3. Consider upgrading to paid plan for production
4. Set up custom domain (optional)
5. Enable SSL/TLS (automatic on Render)
