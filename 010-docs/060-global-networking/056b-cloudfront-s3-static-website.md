# Lab Guide: CloudFront with S3 for Static Website Hosting

## Topic: Deploying a Global Static Website with CloudFront and S3

### Prerequisites
- AWS account with permissions for S3, CloudFront, and Route 53
- Basic understanding of web hosting and CDN concepts
- Text editor for creating HTML files
- Optional: Custom domain name for advanced configuration

### Learning Objectives
By the end of this lab, you will be able to:
- Create and configure an S3 bucket for static website hosting
- Deploy a CloudFront distribution to serve S3 content globally
- Configure Origin Access Control (OAC) for secure S3 access
- Implement HTTPS and custom domain names
- Optimize caching behaviors for performance
- Monitor and troubleshoot CloudFront distributions

### Architecture Overview
This lab builds a globally distributed static website using:
- **S3 Bucket**: Stores static website files (HTML, CSS, JS, images)
- **CloudFront Distribution**: CDN that caches content at edge locations worldwide
- **Origin Access Control (OAC)**: Restricts S3 access to only CloudFront
- **Optional Route 53**: Custom domain mapping with SSL/TLS

**Traffic Flow**: User → CloudFront Edge Location → S3 Origin (if cache miss)

### Lab Duration
Estimated time: 45-60 minutes

---

## Lab Steps

### Step 1: Create S3 Bucket for Static Content

**Objective**: Set up an S3 bucket to store website files

**Theory**: S3 can host static websites, but when paired with CloudFront, we get global distribution, HTTPS support, better performance, and DDoS protection. We'll keep the bucket private and use CloudFront's Origin Access Control.

**AWS Console Steps**:
1. Navigate to **S3 Console** → Click **Create bucket**
2. **Bucket name**: `cloudfront-static-lab-[your-initials]-[random-number]`
   - Example: `cloudfront-static-lab-js-12345`
3. **Region**: Choose your preferred region (e.g., us-east-1)
4. **Block Public Access settings**: Leave all checked (bucket will stay private)
5. **Bucket Versioning**: Enable (optional, useful for rollbacks)
6. **Encryption**: Enable with SSE-S3
7. Click **Create bucket**

**CLI Commands**:
```bash
# Create S3 bucket
aws s3 mb s3://cloudfront-static-lab-js-12345 --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket cloudfront-static-lab-js-12345 \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket cloudfront-static-lab-js-12345 \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

**CLI Verification**:
```bash
# Verify bucket exists
aws s3 ls | grep cloudfront-static-lab

# Check bucket versioning status
aws s3api get-bucket-versioning --bucket cloudfront-static-lab-js-12345

# Check encryption configuration
aws s3api get-bucket-encryption --bucket cloudfront-static-lab-js-12345

# Check public access block settings
aws s3api get-public-access-block --bucket cloudfront-static-lab-js-12345
```

**Expected Output**:
```json
// Versioning
{
    "Status": "Enabled"
}

// Encryption
{
    "ServerSideEncryptionConfiguration": {
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "AES256"
            }
        }]
    }
}
```

**Verification Checklist**:
- ✅ Bucket appears in S3 console
- ✅ Public access is blocked (all 4 settings enabled)
- ✅ Versioning shows "Enabled"
- ✅ Encryption shows "SSE-S3"

---

### Step 2: Upload Sample Website Content

**Objective**: Create and upload static website files to S3

**Theory**: A static website consists of HTML, CSS, JavaScript, and media files. These files are served as-is without server-side processing.

**Create Sample Files**:

**index.html**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CloudFront Lab - Home</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <header>
        <h1>Welcome to CloudFront Static Website Lab</h1>
        <nav>
            <a href="index.html">Home</a>
            <a href="about.html">About</a>
        </nav>
    </header>
    <main>
        <h2>Global Content Delivery with Amazon CloudFront</h2>
        <p>This website is served from an S3 bucket through CloudFront's global edge network.</p>
        <div class="info-box">
            <h3>Your Connection Info</h3>
            <p id="timestamp"></p>
            <p id="cache-status">Check CloudFront headers for cache status</p>
        </div>
        <img src="images/cloudfront-architecture.png" alt="CloudFront Architecture" style="max-width: 100%;">
    </main>
    <footer>
        <p>Served by Amazon CloudFront | <span id="edge-location">Edge Location: Check Response Headers</span></p>
    </footer>
    <script>
        document.getElementById('timestamp').textContent = 'Page loaded at: ' + new Date().toLocaleString();
    </script>
</body>
</html>
```

**about.html**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CloudFront Lab - About</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <header>
        <h1>About CloudFront</h1>
        <nav>
            <a href="index.html">Home</a>
            <a href="about.html">About</a>
        </nav>
    </header>
    <main>
        <h2>What is Amazon CloudFront?</h2>
        <p>Amazon CloudFront is a fast content delivery network (CDN) service that securely delivers data, videos, applications, and APIs to customers globally with low latency and high transfer speeds.</p>
        <h3>Key Features:</h3>
        <ul>
            <li>Global edge network with 400+ Points of Presence</li>
            <li>Integrated with AWS Shield for DDoS protection</li>
            <li>Custom SSL/TLS certificates</li>
            <li>Real-time metrics and logging</li>
            <li>Lambda@Edge for edge computing</li>
        </ul>
    </main>
    <footer>
        <p>Served by Amazon CloudFront</p>
    </footer>
</body>
</html>
```

**styles.css**:
```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    line-height: 1.6;
    color: #333;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
}

header {
    background: rgba(255, 255, 255, 0.95);
    padding: 1rem 2rem;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

header h1 {
    color: #232f3e;
    margin-bottom: 1rem;
}

nav a {
    margin-right: 1.5rem;
    text-decoration: none;
    color: #ff9900;
    font-weight: bold;
    transition: color 0.3s;
}

nav a:hover {
    color: #ff6600;
}

main {
    max-width: 900px;
    margin: 2rem auto;
    padding: 2rem;
    background: white;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

h2 {
    color: #232f3e;
    margin-bottom: 1rem;
}

.info-box {
    background: #f0f8ff;
    border-left: 4px solid #ff9900;
    padding: 1rem;
    margin: 1.5rem 0;
}

.info-box h3 {
    color: #232f3e;
    margin-bottom: 0.5rem;
}

footer {
    text-align: center;
    padding: 2rem;
    color: white;
    font-size: 0.9rem;
}

ul {
    margin-left: 2rem;
    margin-top: 1rem;
}

li {
    margin-bottom: 0.5rem;
}
```

**error.html**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Not Found</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <main>
        <h2>404 - Page Not Found</h2>
        <p>The page you're looking for doesn't exist.</p>
        <p><a href="index.html">Return to Home</a></p>
    </main>
</body>
</html>
```

**Upload Files to S3**:

**AWS Console Steps**:
1. Open your S3 bucket
2. Click **Upload** → **Add files**
3. Select `index.html`, `about.html`, `styles.css`, `error.html`
4. Click **Upload**
5. Create folder **images** and upload a sample image (optional)

**CLI Commands**:
```bash
# Upload all files
aws s3 cp index.html s3://cloudfront-static-lab-js-12345/
aws s3 cp about.html s3://cloudfront-static-lab-js-12345/
aws s3 cp styles.css s3://cloudfront-static-lab-js-12345/
aws s3 cp error.html s3://cloudfront-static-lab-js-12345/

# Upload entire directory (if you have files in a local folder)
aws s3 sync ./website/ s3://cloudfront-static-lab-js-12345/
```

**CLI Verification**:
```bash
# List all files in bucket
aws s3 ls s3://cloudfront-static-lab-js-12345/

# Verify specific files were uploaded
aws s3 ls s3://cloudfront-static-lab-js-12345/ --recursive

# Check object count
aws s3 ls s3://cloudfront-static-lab-js-12345/ --recursive | wc -l
```

**Expected Output**:
```
2024-01-15 10:30:45       1234 index.html
2024-01-15 10:30:46       2456 about.html
2024-01-15 10:30:47       1890 styles.css
2024-01-15 10:30:48        678 error.html
```

**Verification Checklist**:
- ✅ Files visible in S3 console
- ✅ File permissions show "Bucket and objects not public"
- ✅ All 4 files uploaded successfully

---

### Step 3: Create CloudFront Distribution

**Objective**: Create a CloudFront distribution to serve S3 content globally

**Theory**: CloudFront caches content at edge locations closest to users. The first request fetches from S3 (origin), subsequent requests are served from cache. Origin Access Control ensures only CloudFront can access the S3 bucket.

**AWS Console Steps**:
1. Navigate to **CloudFront Console** → Click **Create distribution**
2. **Origin Settings**:
   - **Origin domain**: Select your S3 bucket from dropdown
   - **Name**: Auto-populated (keep default)
   - **Origin path**: Leave blank
   - **Origin access**: Select **Origin access control settings (recommended)**
   - Click **Create control setting**:
     - **Name**: `S3-OAC-static-website`
     - **Signing behavior**: Sign requests (recommended)
     - Click **Create**

3. **Default Cache Behavior**:
   - **Viewer protocol policy**: Redirect HTTP to HTTPS
   - **Allowed HTTP methods**: GET, HEAD
   - **Cache policy**: CachingOptimized (recommended)
   - **Origin request policy**: None

4. **Settings**:
   - **Price class**: Use all edge locations (best performance)
   - **Alternate domain names (CNAMEs)**: Leave blank for now
   - **Custom SSL certificate**: Leave default (CloudFront certificate)
   - **Default root object**: `index.html`
   - **Standard logging**: Off (or enable to S3 for monitoring)

5. Click **Create distribution**

6. **Important**: Copy the S3 bucket policy shown in the banner and apply it:
   - Click **Copy policy**
   - Go to S3 bucket → **Permissions** → **Bucket policy** → Paste and **Save**

**CLI Commands**:
```bash
# Create Origin Access Control
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "S3-OAC-static-website",
    "Description": "OAC for static website S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' > oac-output.json

# Extract OAC ID
OAC_ID=$(cat oac-output.json | grep -o '"Id": "[^"]*' | grep -o '[^"]*$')

# Create distribution config file
cat > distribution-config.json << 'EOF'
{
  "CallerReference": "static-website-lab-2024",
  "Comment": "CloudFront distribution for S3 static website",
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-cloudfront-static-lab",
        "DomainName": "cloudfront-static-lab-js-12345.s3.amazonaws.com",
        "OriginAccessControlId": "YOUR_OAC_ID",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-cloudfront-static-lab",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"],
      "CachedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"]
      }
    },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true,
    "MinTTL": 0
  },
  "Enabled": true
}
EOF

# Replace OAC_ID in config
sed -i "s/YOUR_OAC_ID/$OAC_ID/" distribution-config.json

# Create distribution
aws cloudfront create-distribution --distribution-config file://distribution-config.json > distribution-output.json

# Get distribution domain name
DIST_DOMAIN=$(cat distribution-output.json | grep -o '"DomainName": "[^"]*' | grep -o '[^"]*$' | head -1)
echo "Distribution Domain: $DIST_DOMAIN"
```

**Update S3 Bucket Policy**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::cloudfront-static-lab-js-12345/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
                }
            }
        }
    ]
}
```

**CLI Verification**:
```bash
# List CloudFront distributions
aws cloudfront list-distributions --query 'DistributionList.Items[*].[Id,DomainName,Status,Origins.Items[0].DomainName]' --output table

# Get specific distribution details
aws cloudfront get-distribution --id YOUR_DISTRIBUTION_ID --query 'Distribution.[Id,DomainName,Status,DistributionConfig.DefaultRootObject]' --output table

# Check distribution status
aws cloudfront get-distribution --id YOUR_DISTRIBUTION_ID --query 'Distribution.Status' --output text

# Verify S3 bucket policy
aws s3api get-bucket-policy --bucket cloudfront-static-lab-js-12345 --query Policy --output text | jq .
```

**Expected Output**:
```
# Status should show "InProgress" then "Deployed"
InProgress  (wait 5-15 minutes)
Deployed    (ready to use)

# Distribution domain
d1234567890abc.cloudfront.net
```

**Verification Checklist**:
- ✅ Distribution status shows "InProgress" or "Deployed"
- ✅ Distribution domain name is visible (e.g., `d1234abcd5678.cloudfront.net`)
- ✅ S3 bucket policy is updated with CloudFront service principal
- ✅ Origin Access Control (OAC) is configured
- ✅ Default root object is set to `index.html`

---

### Step 4: Test the CloudFront Distribution

**Objective**: Verify the website is accessible through CloudFront

**Theory**: Once deployed, CloudFront serves content from edge locations. Initial requests are cache misses (X-Cache: Miss from cloudfront), subsequent requests are cache hits (X-Cache: Hit from cloudfront).

**Wait for Deployment**:
- Check **Status** column in CloudFront console
- Wait until it changes from "Deploying" to "Enabled" (5-15 minutes)

**Testing Steps**:

1. **Access the Website**:
   - Copy the **Distribution domain name** (e.g., `d1234abcd5678.cloudfront.net`)
   - Open in browser: `https://d1234abcd5678.cloudfront.net`
   - You should see the homepage

2. **Test Navigation**:
   - Click "About" link → `about.html` should load
   - Try invalid URL → Should show default S3 error (we'll fix this next)

3. **Check CloudFront Headers** (Developer Tools → Network Tab):
   ```
   HTTP/2 200
   x-cache: Miss from cloudfront
   x-amz-cf-pop: IAD89-C1  (Edge location code)
   x-amz-cf-id: [unique request ID]
   ```

4. **Test Cache Hit** (Refresh page):
   ```
   x-cache: Hit from cloudfront
   age: 45  (seconds since cached)
   ```

**CLI Testing**:
```bash
# Test with curl
curl -I https://d1234abcd5678.cloudfront.net

# Check cache status
curl -s -I https://d1234abcd5678.cloudfront.net | grep -i "x-cache"

# Test from different locations (using web tools)
# Check latency from multiple regions
```

**CLI Verification**:
```bash
# Test website accessibility
curl -I https://d1234abcd5678.cloudfront.net

# Check CloudFront headers
curl -s -I https://d1234abcd5678.cloudfront.net | grep -E "x-cache|x-amz-cf-pop|x-amz-cf-id"

# Test cache behavior (run twice)
echo "First request (cache miss):"
curl -s -I https://d1234abcd5678.cloudfront.net | grep x-cache
sleep 2
echo "Second request (cache hit):"
curl -s -I https://d1234abcd5678.cloudfront.net | grep x-cache

# Test HTTPS redirect
curl -I http://d1234abcd5678.cloudfront.net

# Download full page content
curl -s https://d1234abcd5678.cloudfront.net | head -20

# Test different pages
curl -I https://d1234abcd5678.cloudfront.net/about.html
curl -I https://d1234abcd5678.cloudfront.net/styles.css
```

**Expected Output**:
```
HTTP/2 200
x-cache: Miss from cloudfront          (first request)
x-amz-cf-pop: IAD89-C1                  (edge location)
x-amz-cf-id: abc123...                  (request ID)

# Second request:
x-cache: Hit from cloudfront            (cached)
age: 5                                   (seconds since cached)
```

**Verification Checklist**:
- ✅ Website loads successfully over HTTPS (200 OK)
- ✅ CloudFront headers present (x-cache, x-amz-cf-pop)
- ✅ HTTP redirects to HTTPS (301 or 302)
- ✅ Second request shows cache hit
- ✅ All pages accessible (index.html, about.html)

---

### Step 5: Configure Custom Error Pages

**Objective**: Set up custom error responses for better user experience

**Theory**: By default, S3 returns XML error responses. CloudFront can intercept these and return custom HTML pages instead.

**AWS Console Steps**:
1. Go to **CloudFront** → Your distribution → **Error pages** tab
2. Click **Create custom error response**
3. **Error 404**:
   - **HTTP error code**: 404
   - **Customize error response**: Yes
   - **Response page path**: `/error.html`
   - **HTTP response code**: 404 (or 200 for SPA)
   - **Cache TTL**: 300 seconds
4. Click **Create custom error response**
5. Repeat for **403 errors** (same settings)

**CLI Commands**:
```bash
# Get current distribution config
aws cloudfront get-distribution-config --id YOUR_DISTRIBUTION_ID > dist-config.json

# Extract ETag
ETAG=$(cat dist-config.json | grep -o '"ETag": "[^"]*' | grep -o '[^"]*$')

# Add custom error responses to config (manually edit dist-config.json)
# Then update distribution:
aws cloudfront update-distribution \
  --id YOUR_DISTRIBUTION_ID \
  --if-match $ETAG \
  --distribution-config file://updated-dist-config.json
```

**Testing**:
```bash
# Test 404 error
curl https://d1234abcd5678.cloudfront.net/nonexistent.html
# Should return custom error.html
```

**CLI Verification**:
```bash
# Test 404 error page
curl -I https://d1234abcd5678.cloudfront.net/nonexistent.html

# Get full error page content
curl -s https://d1234abcd5678.cloudfront.net/does-not-exist.html

# Verify error response has correct status code
curl -s -o /dev/null -w "%{http_code}\n" https://d1234abcd5678.cloudfront.net/missing.html

# Check custom error response configuration
aws cloudfront get-distribution-config --id YOUR_DISTRIBUTION_ID --query 'DistributionConfig.CustomErrorResponses' --output json
```

**Expected Output**:
```json
{
    "Quantity": 2,
    "Items": [
        {
            "ErrorCode": 404,
            "ResponsePagePath": "/error.html",
            "ResponseCode": "404",
            "ErrorCachingMinTTL": 300
        },
        {
            "ErrorCode": 403,
            "ResponsePagePath": "/error.html",
            "ResponseCode": "404",
            "ErrorCachingMinTTL": 300
        }
    ]
}
```

**Verification Checklist**:
- ✅ Non-existent pages return custom error.html content
- ✅ Error page has HTTP 404 status code
- ✅ Error page has CloudFront caching headers
- ✅ Both 403 and 404 errors configured

---

### Step 6: Configure Cache Behaviors and Invalidations

**Objective**: Optimize caching and learn to invalidate cached content

**Theory**: Different content types have different caching needs. Static assets (CSS, JS, images) can cache longer; HTML might need shorter TTLs. Invalidations clear cached content when you update files.

**Create Cache Behavior for Static Assets**:

**AWS Console Steps**:
1. Go to **CloudFront** → Distribution → **Behaviors** tab
2. Click **Create behavior**
3. **Path pattern**: `*.css`
   - **Cache policy**: CachingOptimized
   - **TTL**: Min 3600, Max 86400, Default 86400
4. Create another for `*.js` (same settings)
5. Create another for `/images/*` (longer TTL: 604800 = 7 days)

**Update Content and Invalidate Cache**:

1. **Modify index.html**:
   - Update some text
   - Upload to S3 (overwrite existing file)

2. **Create Invalidation**:
   - **CloudFront** → Distribution → **Invalidations** tab
   - Click **Create invalidation**
   - **Object paths**:
     ```
     /index.html
     /about.html
     ```
   - Click **Create invalidation**
   - Wait ~30 seconds for completion

**CLI Commands**:
```bash
# Upload updated file
aws s3 cp index.html s3://cloudfront-static-lab-js-12345/

# Create invalidation
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/index.html" "/about.html"

# Invalidate everything (use sparingly - can incur charges)
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

**CLI Verification**:
```bash
# List all invalidations
aws cloudfront list-invalidations --distribution-id YOUR_DISTRIBUTION_ID --output table

# Check invalidation status
aws cloudfront get-invalidation --distribution-id YOUR_DISTRIBUTION_ID --id INVALIDATION_ID --query 'Invalidation.Status' --output text

# Get invalidation details
aws cloudfront get-invalidation --distribution-id YOUR_DISTRIBUTION_ID --id INVALIDATION_ID --output json

# Verify content was updated (check for cache miss)
curl -s -I https://d1234abcd5678.cloudfront.net/index.html | grep x-cache

# View cache behaviors
aws cloudfront get-distribution-config --id YOUR_DISTRIBUTION_ID --query 'DistributionConfig.CacheBehaviors' --output json
```

**Expected Output**:
```
# Invalidation status
Completed

# After invalidation, first request should be cache miss
x-cache: Miss from cloudfront
```

**Verification Checklist**:
- ✅ Invalidation status shows "Completed"
- ✅ Website shows updated content (check browser)
- ✅ First request after invalidation shows "Miss from cloudfront"
- ✅ Cache behaviors configured for *.css, *.js, /images/*
- ✅ Updated files are now visible on the website

---

### Step 7: Enable Logging and Monitoring

**Objective**: Set up logging and CloudWatch metrics for monitoring

**Theory**: CloudFront provides access logs (detailed request logs) and real-time metrics in CloudWatch for monitoring performance and troubleshooting.

**Enable Access Logging**:

**AWS Console Steps**:
1. Create S3 bucket for logs: `cloudfront-logs-js-12345`
2. **CloudFront** → Distribution → **General** → Edit
3. **Standard logging**: On
4. **S3 bucket**: `cloudfront-logs-js-12345.s3.amazonaws.com`
5. **Log prefix**: `static-website/`
6. **Cookie logging**: Off
7. Save changes

**View Metrics in CloudWatch**:
1. **CloudWatch Console** → **Metrics** → **CloudFront**
2. View metrics:
   - **Requests**: Total requests per distribution
   - **BytesDownloaded**: Data transfer
   - **ErrorRate**: 4xx and 5xx errors
   - **CacheHitRate**: Percentage of cache hits

**Create CloudWatch Alarm**:
1. **CloudWatch** → **Alarms** → **Create alarm**
2. **Select metric**: CloudFront → Per-Distribution Metrics → 4xxErrorRate
3. **Conditions**: Greater than 5% for 2 consecutive periods
4. **Actions**: SNS notification (optional)

**CLI Commands**:
```bash
# Enable logging
aws cloudfront update-distribution --id YOUR_DISTRIBUTION_ID \
  --distribution-config '{
    "Logging": {
      "Enabled": true,
      "IncludeCookies": false,
      "Bucket": "cloudfront-logs-js-12345.s3.amazonaws.com",
      "Prefix": "static-website/"
    }
  }'

# Query CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name Requests \
  --dimensions Name=DistributionId,Value=YOUR_DISTRIBUTION_ID \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z \
  --period 3600 \
  --statistics Sum
```

**CLI Verification**:
```bash
# Check logging configuration
aws cloudfront get-distribution-config --id YOUR_DISTRIBUTION_ID --query 'DistributionConfig.Logging' --output json

# List log files in S3 (after waiting 15-60 minutes)
aws s3 ls s3://cloudfront-logs-js-12345/static-website/

# View recent log file
aws s3 cp s3://cloudfront-logs-js-12345/static-website/E1234567890ABC.2024-01-15-10.a1b2c3d4.gz - | gunzip | head -20

# Get CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name Requests \
  --dimensions Name=DistributionId,Value=YOUR_DISTRIBUTION_ID \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Sum

# List available CloudWatch metrics
aws cloudwatch list-metrics --namespace AWS/CloudFront --dimensions Name=DistributionId,Value=YOUR_DISTRIBUTION_ID

# Check alarms
aws cloudwatch describe-alarms --alarm-name-prefix cloudfront
```

**Expected Output**:
```json
// Logging config
{
    "Enabled": true,
    "IncludeCookies": false,
    "Bucket": "cloudfront-logs-js-12345.s3.amazonaws.com",
    "Prefix": "static-website/"
}

// Available metrics
- Requests
- BytesDownloaded
- BytesUploaded
- 4xxErrorRate
- 5xxErrorRate
- TotalErrorRate
```

**Verification Checklist**:
- ✅ Logging is enabled in distribution config
- ✅ Log files appear in S3 bucket (after 15-60 minutes)
- ✅ CloudWatch shows metrics for your distribution
- ✅ Metrics show request counts, bytes transferred
- ✅ Alarms are configured (if created)

---

### Step 8: Add Custom Domain with SSL/TLS (Optional)

**Objective**: Configure a custom domain name with HTTPS

**Prerequisites**: You own a domain name

**Theory**: CloudFront provides a default domain (`*.cloudfront.net`), but for production, you'll want a custom domain with your own SSL certificate. AWS Certificate Manager (ACM) provides free SSL certificates.

**Request SSL Certificate**:

**AWS Console Steps**:
1. Go to **AWS Certificate Manager** (ACM) in **us-east-1 region** (required for CloudFront)
2. Click **Request certificate**
3. **Certificate type**: Request a public certificate
4. **Domain names**:
   - `www.yourdomain.com`
   - `yourdomain.com` (optional, for apex domain)
5. **Validation method**: DNS validation (recommended)
6. Click **Request**
7. **Add CNAME records** to your DNS (Route 53 or external):
   - ACM provides CNAME name and value
   - Add to your DNS provider
8. Wait for validation (a few minutes to hours)

**Configure CloudFront for Custom Domain**:

1. **CloudFront** → Distribution → **General** → Edit
2. **Alternate domain names (CNAMEs)**:
   - Add `www.yourdomain.com`
3. **Custom SSL certificate**: Select your ACM certificate from dropdown
4. **Save changes**
5. Wait for deployment (~5-10 minutes)

**Update DNS to Point to CloudFront**:

**Route 53** (if using AWS DNS):
1. **Route 53** → Hosted zones → Your domain
2. **Create record**:
   - **Record name**: `www`
   - **Record type**: A
   - **Alias**: Yes
   - **Route traffic to**: Alias to CloudFront distribution
   - **Choose distribution**: Select your distribution
3. Create record

**External DNS**:
- Create CNAME record: `www` → `d1234abcd5678.cloudfront.net`

**CLI Commands**:
```bash
# Request certificate
aws acm request-certificate \
  --domain-name www.yourdomain.com \
  --validation-method DNS \
  --region us-east-1

# Update CloudFront distribution
aws cloudfront update-distribution \
  --id YOUR_DISTRIBUTION_ID \
  --distribution-config '{
    "Aliases": {
      "Quantity": 1,
      "Items": ["www.yourdomain.com"]
    },
    "ViewerCertificate": {
      "ACMCertificateArn": "arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT_ID",
      "SSLSupportMethod": "sni-only",
      "MinimumProtocolVersion": "TLSv1.2_2021"
    }
  }'

# Create Route 53 record
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "d1234abcd5678.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

**AWS Console Verification Steps**:
1. **Verify ACM Certificate**:
   - Go to **AWS Certificate Manager** (us-east-1)
   - Certificate status should show "Issued"
   - Domains should show both `rapidgrasp.com` and `www.rapidgrasp.com`
   - Validation status: "Success"

2. **Verify CloudFront Configuration**:
   - **CloudFront Console** → Your distribution → **General** tab
   - **Alternate domain names** should show: `rapidgrasp.com`, `www.rapidgrasp.com`
   - **SSL Certificate** should show your custom ACM certificate (not CloudFront default)
   - **Distribution status**: "Deployed" (not "InProgress")

3. **Verify DNS Configuration**:
   - **Route 53 Console** → Hosted zones → `rapidgrasp.com`
   - Should see A record for `rapidgrasp.com` (or `www`) pointing to CloudFront
   - Record type: "A - IPv4 address"
   - Alias: "Yes"
   - Alias target: Your CloudFront distribution domain

4. **Browser Testing**:
   - Open `https://rapidgrasp.com` in browser
   - Click padlock icon → Certificate should show:
     - Issued to: `rapidgrasp.com`
     - Issued by: Amazon
     - Valid dates shown
   - No SSL warnings or errors

**CLI Verification Commands**:
```bash
# 1. Verify ACM certificate status
aws acm list-certificates --region us-east-1 --query 'CertificateSummaryList[?DomainName==`rapidgrasp.com`]' --output table

# Get certificate details
aws acm describe-certificate --certificate-arn YOUR_CERT_ARN --region us-east-1 --query 'Certificate.[DomainName,Status,DomainValidationOptions[0].ValidationStatus]' --output table

# 2. Verify CloudFront has custom domain configured
aws cloudfront get-distribution --id YOUR_DISTRIBUTION_ID --query 'Distribution.DistributionConfig.[Aliases.Items,ViewerCertificate.ACMCertificateArn]' --output json

# 3. Test DNS resolution
dig rapidgrasp.com
dig www.rapidgrasp.com

# Verify DNS points to CloudFront
nslookup rapidgrasp.com
# Should return CloudFront domain: d1234567890abc.cloudfront.net

# 4. Test HTTPS connection
curl -I https://rapidgrasp.com
curl -I https://www.rapidgrasp.com

# Check SSL certificate details
curl -vI https://rapidgrasp.com 2>&1 | grep -E "subject:|issuer:|SSL certificate"

# Test SSL/TLS version
openssl s_client -connect rapidgrasp.com:443 -tls1_2 < /dev/null 2>&1 | grep -E "Protocol|Cipher"

# 5. Verify HTTP to HTTPS redirect
curl -I http://rapidgrasp.com

# 6. Test from multiple locations (using online tools or different servers)
curl -s -I -L https://rapidgrasp.com | grep -E "HTTP|x-cache|x-amz-cf-pop"

# 7. Check Route 53 record
aws route53 list-resource-record-sets --hosted-zone-id YOUR_ZONE_ID --query "ResourceRecordSets[?Name=='rapidgrasp.com.']" --output json

# 8. Verify certificate is attached to distribution
aws cloudfront get-distribution-config --id YOUR_DISTRIBUTION_ID --query 'DistributionConfig.ViewerCertificate' --output json
```

**Expected Output**:
```json
// ACM Certificate
{
    "DomainName": "rapidgrasp.com",
    "Status": "ISSUED",
    "ValidationStatus": "SUCCESS"
}

// CloudFront Aliases
{
    "Aliases": {
        "Items": ["rapidgrasp.com", "www.rapidgrasp.com"]
    },
    "ViewerCertificate": {
        "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc-123...",
        "SSLSupportMethod": "sni-only",
        "MinimumProtocolVersion": "TLSv1.2_2021"
    }
}

// DNS Resolution
rapidgrasp.com.  300  IN  A  [CloudFront IP addresses]

// HTTPS Test
HTTP/2 200
server: CloudFront
x-cache: Hit from cloudfront
```

**Verification Checklist**:
- ✅ ACM certificate status is "Issued"
- ✅ Certificate includes both `rapidgrasp.com` and `www.rapidgrasp.com`
- ✅ CloudFront shows custom domain in Aliases
- ✅ CloudFront shows custom ACM certificate (not default)
- ✅ DNS resolves to CloudFront distribution
- ✅ `https://rapidgrasp.com` loads your website
- ✅ `https://www.rapidgrasp.com` loads your website
- ✅ SSL certificate shows as valid in browser
- ✅ Browser shows secure connection (padlock icon)
- ✅ HTTP automatically redirects to HTTPS
- ✅ No SSL warnings or certificate errors

---

### Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| **403 Forbidden from S3** | S3 bucket policy not updated | Copy CloudFront-provided bucket policy to S3 bucket permissions |
| **Distribution not accessible** | Still deploying | Wait 10-15 minutes for full deployment; status must show "Enabled" |
| **Changes not appearing** | Content cached | Create CloudFront invalidation for updated files |
| **Custom domain not working** | DNS not propagated or SSL cert not validated | Check DNS settings, verify ACM certificate status is "Issued" |
| **404 on subdirectories** | No index.html in subdirectory | Use Lambda@Edge for path rewrites or ensure all paths have index.html |
| **High latency on first load** | Cache miss, fetching from S3 origin | Normal behavior; subsequent requests will be faster (cache hit) |
| **SSL certificate error** | Certificate in wrong region | ACM certificate MUST be in us-east-1 for CloudFront |
| **Access denied after bucket policy** | Typo in ARN or condition | Verify distribution ARN matches condition in bucket policy |

---

### Cleanup

To avoid ongoing charges:

1. **Disable CloudFront distribution**:
   - CloudFront → Distribution → Disable
   - Wait for status to change to "Disabled" (~15 minutes)
   - Then delete the distribution

2. **Delete S3 buckets**:
   - Empty buckets first (delete all objects)
   - Delete `cloudfront-static-lab-js-12345`
   - Delete `cloudfront-logs-js-12345` (if created)

3. **Delete ACM certificate** (if created):
   - Ensure it's not in use by CloudFront
   - ACM → Delete certificate

4. **Remove Route 53 records** (if created):
   - Delete A record for `www.yourdomain.com`

**CLI Cleanup**:
```bash
# Disable distribution
aws cloudfront update-distribution \
  --id YOUR_DISTRIBUTION_ID \
  --distribution-config '{"Enabled": false, ...}'

# Wait for deployment, then delete
aws cloudfront delete-distribution --id YOUR_DISTRIBUTION_ID --if-match ETAG

# Empty and delete S3 buckets
aws s3 rm s3://cloudfront-static-lab-js-12345 --recursive
aws s3 rb s3://cloudfront-static-lab-js-12345

# Delete certificate
aws acm delete-certificate --certificate-arn YOUR_CERT_ARN --region us-east-1
```

**CLI Verification**:
```bash
# Verify distribution is disabled
aws cloudfront get-distribution --id YOUR_DISTRIBUTION_ID --query 'Distribution.DistributionConfig.Enabled' --output text
# Should return: false

# Verify distribution is deleted (should return error)
aws cloudfront get-distribution --id YOUR_DISTRIBUTION_ID 2>&1
# Should return: NoSuchDistribution error

# Verify S3 buckets are empty and deleted
aws s3 ls s3://cloudfront-static-lab-js-12345 2>&1
# Should return: NoSuchBucket error

aws s3 ls | grep cloudfront
# Should return: empty (no results)

# Verify ACM certificate is deleted (if applicable)
aws acm list-certificates --region us-east-1 --query 'CertificateSummaryList[?DomainName==`rapidgrasp.com`]'
# Should return: empty list

# Check Route 53 records are removed
aws route53 list-resource-record-sets --hosted-zone-id YOUR_ZONE_ID --query "ResourceRecordSets[?Name=='rapidgrasp.com.']"
# Should return: only SOA and NS records (no A record)
```

**Expected Output**:
```
# Distribution deleted
An error occurred (NoSuchDistribution) when calling the GetDistribution operation

# Bucket deleted
An error occurred (NoSuchBucket) when calling the ListObjectsV2 operation

# No CloudFront distributions
{
    "DistributionList": {
        "Items": []
    }
}
```

**Verification Checklist**:
- ✅ No CloudFront distributions listed in console
- ✅ S3 buckets deleted (both website and logs bucket)
- ✅ ACM certificate deleted (if created)
- ✅ Route 53 A records removed (if created)
- ✅ No charges in billing dashboard after 24 hours
- ✅ No orphaned resources remaining

---

### Cost Analysis

**Monthly Cost Estimate for CloudFront + S3 Static Website**

#### Assumptions (Small Static Website):
- Website size: 100 MB total content
- Traffic: 10,000 page views/month
- Average page size: 500 KB (HTML, CSS, JS, images)
- Total data transfer: ~5 GB/month
- Region: US/Europe

#### Cost Breakdown:

**1. Amazon S3 Storage:**
- Storage: 0.1 GB × $0.023/GB = **$0.002/month**
- PUT requests: 10 files × $0.005/1,000 = **~$0.00**
- GET requests (cached by CloudFront): **~$0.00**

**2. Amazon CloudFront:**
- Data transfer out: 5 GB × $0.085/GB = **$0.425**
  - *First 1 TB free with AWS Free Tier*
- HTTPS requests: 10,000 × $0.01/10,000 = **$0.001**
  - *First 10M requests free with AWS Free Tier*

**3. AWS Certificate Manager (ACM):**
- SSL/TLS certificate for custom domain = **FREE**

**4. Amazon Route 53 (Optional - Custom Domain):**
- Hosted zone: **$0.50/month**
- DNS queries: 10,000 × $0.40/million = **~$0.004**

#### Total Monthly Cost:

**With AWS Free Tier (First 12 months):**
```
S3 Storage:        $0.00  (under 5 GB free tier)
CloudFront:        $0.00  (under 1 TB + 10M requests free)
ACM Certificate:   $0.00  (always free)
Route 53:          $0.50  (hosted zone + queries)
────────────────────────
TOTAL:            ~$0.50/month
```

**Without AWS Free Tier (After 12 months):**
```
S3 Storage:        $0.002
CloudFront:        $0.426
ACM Certificate:   $0.00
Route 53:          $0.504
────────────────────────
TOTAL:            ~$0.93/month
```

#### Cost Scaling by Traffic Level:

| Traffic Level | Page Views/Month | Data Transfer | S3 Cost | CloudFront Cost | Route 53 | **Total/Month** |
|---------------|------------------|---------------|---------|-----------------|----------|-----------------|
| **Low** | 10,000 | 5 GB | $0.002 | $0.43 | $0.50 | **$0.93** |
| **Medium** | 100,000 | 50 GB | $0.002 | $4.25 | $0.50 | **$4.75** |
| **High** | 1,000,000 | 500 GB | $0.005 | $42.50 | $0.50 | **$43.01** |
| **Very High** | 10,000,000 | 5 TB | $0.015 | $425.00 | $0.50 | **$425.52** |

*Note: Prices assume US/Europe regions and no Free Tier. Actual costs may vary by region.*

#### Cost Optimization Strategies:

**1. Maximize Free Tier Benefits:**
- AWS Free Tier includes 1 TB CloudFront data transfer
- 10 million HTTPS requests per month
- 5 GB S3 storage (first 12 months)
- Perfect for development and low-traffic sites

**2. Enable Compression:**
- CloudFront automatic compression reduces transfer by 60-80%
- Enable in CloudFront distribution settings
- Works for HTML, CSS, JS, JSON, XML

**3. Optimize Content:**
- Compress images (WebP format saves 25-35% vs JPEG)
- Minify CSS/JS files
- Use responsive images with srcset
- Remove unused code and assets

**4. Configure Longer Cache TTLs:**
- Static assets (CSS/JS/images): 1-7 days
- Reduces origin requests to S3
- Lower S3 GET request charges
- Use versioned file names (e.g., `app.v2.css`) instead of invalidations

**5. Use CloudFront Price Classes:**
- **All edge locations**: Best performance, highest cost
- **Price Class 200**: Excludes most expensive regions, moderate cost
- **Price Class 100**: US, Europe, Israel only, lowest cost
- Can save 15-30% on data transfer

**6. Avoid Invalidation Costs:**
- First 1,000 invalidation paths free/month
- $0.005 per path after that
- Use versioned file names instead: `style.v2.css`, `app.123abc.js`
- Automate versioning in build process

**7. Monitor and Analyze:**
- Use CloudWatch metrics to track usage
- Set up billing alarms
- Review CloudFront reports monthly
- Identify and optimize high-traffic assets

**8. Regional Considerations:**
- US/Europe data transfer: $0.085/GB
- Asia Pacific: $0.140/GB
- India: $0.170/GB
- Consider price classes to exclude expensive regions

#### Cost Comparison with Alternatives:

| Solution | Monthly Cost (10K views) | Monthly Cost (100K views) | HTTPS | CDN |
|----------|--------------------------|---------------------------|-------|-----|
| **S3 + CloudFront** | $0.93 | $4.75 | ✅ Free | ✅ Global |
| **S3 Static Website Only** | $0.10 | $1.00 | ❌ No | ❌ No |
| **EC2 t3.micro + NGINX** | $8.50 | $8.50 | ✅ ACM | ❌ No |
| **Shared Hosting** | $5-15 | $5-15 | ✅ Varies | ❌ Limited |
| **Netlify Free Tier** | $0 | $0 | ✅ Free | ✅ Yes |
| **Vercel Free Tier** | $0 | $0 | ✅ Free | ✅ Yes |

**Recommendation:** For personal blogs and small business sites with <100K monthly visitors, CloudFront + S3 offers excellent value with professional features (global CDN, HTTPS, custom domain) at <$5/month.

#### Hidden Costs to Watch:

1. **Invalidation Overuse**: Stick to 1,000 paths/month or use versioning
2. **Field-Level Encryption**: Additional charges if enabled
3. **Lambda@Edge**: $0.60 per million requests + compute time
4. **Real-Time Logs**: Additional CloudWatch Logs charges
5. **Origin Shield**: $0.01/10K requests (optional caching layer)
6. **Reserved Capacity**: Commit to usage for discounts (enterprise only)

#### Billing Examples:

**Example 1: Personal Blog (rapidgrasp.com)**
- 5,000 monthly visitors
- 2 GB data transfer
- **Cost: $0.50-$0.70/month** (mostly Route 53)

**Example 2: Small Business Website**
- 50,000 monthly visitors
- 25 GB data transfer
- **Cost: $2.50-$3.00/month**

**Example 3: Popular Documentation Site**
- 500,000 monthly visitors
- 250 GB data transfer
- **Cost: $21-$22/month**

**For rapidgrasp.com with typical low-medium traffic, expect $0.50-$5/month.**

---

### Key Takeaways

- **S3 + CloudFront = Global Static Hosting**: S3 stores files, CloudFront distributes them globally with low latency
- **Origin Access Control (OAC)**: Secures S3 bucket to only allow CloudFront access (replaces deprecated OAI)
- **HTTPS by Default**: CloudFront provides free SSL/TLS with default domain; use ACM for custom domains
- **Caching is Key**: Understand TTL, cache behaviors, and invalidations for optimal performance
- **Edge Locations**: CloudFront has 400+ edge locations worldwide for low-latency content delivery
- **Cost Optimization**: Longer cache TTLs reduce origin requests and data transfer costs
- **Monitoring**: Use CloudWatch metrics and access logs to monitor performance and troubleshoot issues
- **Custom Domains**: Requires ACM certificate in us-east-1 and DNS configuration (Route 53 or external)

---

### Additional Resources

- [AWS CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [Using Amazon S3 as Origin for CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html)
- [CloudFront Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [AWS Certificate Manager](https://docs.aws.amazon.com/acm/)
- [CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/)
- [Lambda@Edge for Advanced Use Cases](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html)

---

### Exam Tips

**For AWS Networking Specialty Exam**:

- **OAC vs OAI**: Origin Access Control (OAC) is the modern replacement for Origin Access Identity (OAI); supports all S3 features including SSE-KMS
- **Regional Edge Caches**: Intermediate caching layer between edge locations and origin (less frequently accessed content)
- **Price Classes**: Three price classes determine which edge locations are used (all, 200, 100); affects both cost and performance
- **Signed URLs/Cookies**: Use for private content; CloudFront verifies signatures before serving
- **Field-Level Encryption**: Encrypts specific form fields at edge, decrypted only by origin application
- **Lambda@Edge**: Run code at edge locations for request/response manipulation; four trigger points: viewer request/response, origin request/response
- **CloudFront vs S3 Transfer Acceleration**: Both use edge locations, but different use cases—CloudFront for content delivery, S3 TA for faster uploads
- **Cache Invalidations**: Free up to 1,000 paths per month; charged $0.005 per path after; use versioned file names to avoid invalidations
- **SSL/TLS**: Three options—Default CloudFront certificate (*.cloudfront.net), ACM certificate (free), Custom certificate (imported)
- **Origin Failover**: Configure secondary origin for high availability; CloudFront automatically fails over on specific error codes

**Common Exam Scenarios**:
- **Static website with global distribution** → S3 + CloudFront
- **Need to secure S3 content** → CloudFront with OAC, no public S3 access
- **Custom domain with HTTPS** → ACM certificate in us-east-1 + CloudFront CNAME
- **Frequently changing content** → Lower TTL or use versioned file names
- **Private content** → CloudFront signed URLs/cookies
- **Reduce origin load** → Optimize cache behaviors and increase TTLs
- **DDoS protection** → CloudFront is integrated with AWS Shield Standard (automatic, free)
