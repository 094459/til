### Today I learned (TIL)

**October, 19th**

When setting up Amazon Cognito to use a custom domain (specifically, your own) you need to make sure that you use the AWS Certificate Manager (ACM) in us-east-1. [This blog post](https://niftytechie.blog/aws/cognito/custom-domain-setup/) captured my issue perfectly.

After creating the certificate in us-east-1, I was still not presented with the drop down option as per the blog post. I was doing this in eu-central-1. I switched to eu-west-1, and the option came up immediately, so it was probably just a delay in the cloudfront distribution being pushed out to all the regions. After about 20 minutes, refreshing the page and the certificate appeared.


**October, 12th**

I am working through an interesting blog post that uses Amazon Cognito and Microsoft Active Directory via ADFS to authenticate users via SAML. I needed to quickly put together a setup so I could test this out. I followed this blog post [How do I set up AD FS on an Amazon EC2 Windows instance to work with federation for an Amazon Cognito user pool?](https://aws.amazon.com/premiumsupport/knowledge-center/cognito-ad-fs-windows-instance/) to build the Windows Server running Active Directory and ADFS. Some additionol things that I did were:

* using my own DNS, I registered the EC2 instance (via the Elastic IP) via a specific domain (I used adfs in my case)
* when integrating into the AWS SSO, you had to create the users in AWS SSO with their email address (e.g. bob@example.com) as stored in Active Directory

**August,2nd**

Wanted a way to get the traffic from my GitHub repos, with the minimum of fuss and stuff to manage. Decided that writing a Lambda function was the way to go, accessing the GitHub API. 

This function is invoked via a cron (every hour) to output GitHub traffic stats which I then use CloudWatch Log Insights to dashboard and get useful info. You need to create your GitHub username/token as Environment Variables (defined in the code as GITHUB_USER and GITHUB_TOKEN) within the Lambda console.

```
# based from the script at https://github.com/AnthonyBloomer/github-traffic-insights
# changed so that it outputs just to use Cloudwatch Insights to track GitHub Traffic

import json
import os
from urllib import request


def record_custom_event(event):
    print(json.dumps(event).encode("utf-8"))


def api_req(request_url):
    token = "token " + os.getenv("GITHUB_TOKEN")
    req = request.Request(request_url, headers={"Accept" :"application/vnd.github.v3+json" , "Authorization": token })
    response = request.urlopen(req)
    return json.load(response)


def lambda_handler(event, context):
    base_url = "https://api.github.com"
    user = os.getenv("GITHUB_USERNAME")
    print("Getting repositories for %s" % user)
    req = api_req("%s/users/%s/repos" % (base_url, user))

    ur = [(r["name"]) for r in req]
    print("Found %s repositories" % len(ur))

    print(" ")

    for repo_name in ur:

        print("Getting referral data for %s" % repo_name)
        referrals = api_req("%s/repos/%s/%s/traffic/popular/referrers" % (base_url, user, repo_name))
        if len(referrals) > 0:
            for ref in referrals:
                referred = {
                    "eventType": "Referral",
                    "repo_name": repo_name,
                    "referrer": ref["referrer"],
                    "count": ref["count"],
                    "uniques": ref["uniques"],
                }
                print(referred)
                record_custom_event(referred)

        print("Getting top referral path data for %s" % repo_name)

        paths = api_req("%s/repos/%s/%s/traffic/popular/paths" % (base_url, user, repo_name))
        if len(paths) > 0:
            for ref in paths:
                paths = {
                    "eventType": "ReferralPath",
                    "repo_name": repo_name,
                    "path": ref["path"],
                    "title": ref["title"],
                    "count": ref["count"],
                    "uniques": ref["uniques"],
                }
                print(paths)
                record_custom_event(paths)

        print("Getting views for %s" % repo_name)
        req = api_req("%s/repos/%s/%s/traffic/views" % (base_url, user, repo_name))
        if req["count"] > 0 or req["uniques"] > 0:
            viewed = {
                "eventType": "View",
                "repo_name": repo_name,
                "count": req["count"],
                "uniques": req["uniques"],
            }
            print(viewed)
            record_custom_event(viewed)

        print("Getting clones for %s" % repo_name)
        req = api_req("%s/repos/%s/%s/traffic/clones" % (base_url, user, repo_name))
        if req["count"] > 0 or req["uniques"] > 0:
            cloned = {
                "eventType": "Clone",
                "repo_name": repo_name,
                "count": req["count"],
                "uniques": req["uniques"],
            }
            record_custom_event(cloned)
        print(" ")
```

Once enabled via an hourly cron, it will generate output in CloudWatch which I can then create a dashboard using a filter like:

*Get daily totals across all repos*
```
fields @timestamp, @message
| filter eventType = 'View'
| stats max(count) as total_user_views by bin(1d) as day
```
*Get total view breakdown by repo*
```
fields @timestamp, @message
| filter eventType='View'
| sort @timestamp, repo_name
| stats max(count) as views by repo_name
```
*Get total clones by repo*
```
fields @timestamp, @message
| filter eventType='Clone'
| sort @timestamp, repo_name
| stats max(count) as views by repo_name
```

