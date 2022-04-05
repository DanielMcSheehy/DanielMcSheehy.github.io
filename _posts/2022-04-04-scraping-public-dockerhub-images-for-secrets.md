---
layout: post
title: Let customers install your integrations using Terraform
description: How to let customers manage permissions and third party integrations with Terraform
summary: 
tags: [terraform]
---
After browsing [dockerhub](https://dockerhub.com), I noticed a section called "recently updated". 
This section includes public images for hello world projects, commercial software, and occasionally
public docker images that should have been private. 

Leaking a public copy of a private image allows anyone to download the image and view it's contents. 
There could be sensitive material inside the leak, such as compiled code binaries and possibly 
credentials/secrets. 

I decided to do an experiment, and figure out a way to download a new public docker images, and scan it for anything intresting. 
This would be tedious to do manually, so I built a scanner that automatically pulls in images, scans the contents/environment variables 
for secrets. The ones below were of particular intrest to me. 
- "Slack Token"
- "RSA private key"
- "SSH (OPENSSH) private key"
- "SSH (DSA) private key"
- "SSH (EC) private key"
- "PGP private key block"
- "Facebook Oauth"
- "Twitter Oauth"
- "GitHub"
- "Google Oauth"
- "AWS API Key"
- "Heroku API Key"
- "Generic Secret"
- "Generic API Key"
- "Slack Webhook"
- "Google (GCP)"
- "Twilio API Key"
- "Password in URL"

Using Rust, I found a pretty good regex library that could find the above given text. 

```rust 
const RULES: &[(&str, &str)] = &[
        ("Slack Token", "(xox[p|b|o|a]-[0-9]{12}-[0-9]{12}-[0-9]{12}-[a-z0-9]{32})"),
        ("RSA private key", "-----BEGIN RSA PRIVATE KEY-----"),
        ("SSH (OPENSSH) private key", "-----BEGIN OPENSSH PRIVATE KEY-----"),
        ("SSH (DSA) private key", "-----BEGIN DSA PRIVATE KEY-----"),
        ("SSH (EC) private key", "-----BEGIN EC PRIVATE KEY-----"),
        ("PGP private key block", "-----BEGIN PGP PRIVATE KEY BLOCK-----"),
        ("Facebook Oauth", "[f|F][a|A][c|C][e|E][b|B][o|O][o|O][k|K].{0,30}['\"\\s][0-9a-f]{32}['\"\\s]"),
        ("Twitter Oauth", "[t|T][w|W][i|I][t|T][t|T][e|E][r|R].{0,30}['\"\\s][0-9a-zA-Z]{35,44}['\"\\s]"),
        ("GitHub", "[g|G][i|I][t|T][h|H][u|U][b|B].{0,30}['\"\\s][0-9a-zA-Z]{35,40}['\"\\s]"),
        ("Google Oauth", "(\"client_secret\":\"[a-zA-Z0-9-_]{24}\")"),
        ("AWS API Key", "AKIA[0-9A-Z]{16}"),
        ("Heroku API Key", "[h|H][e|E][r|R][o|O][k|K][u|U].{0,30}[0-9A-F]{8}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{12}"),
        ("Generic Secret", "[s|S][e|E][c|C][r|R][e|E][t|T].{0,30}['\"\\s][0-9a-zA-Z]{32,45}['\"\\s]"),
        ("Generic API Key", "[a|A][p|P][i|I][_]?[k|K][e|E][y|Y].{0,30}['\"\\s][0-9a-zA-Z]{32,45}['\"\\s]"),
        ("Slack Webhook", "https://hooks.slack.com/services/T[a-zA-Z0-9_]{8}/B[a-zA-Z0-9_]{8}/[a-zA-Z0-9_]{24}"),
        ("Google (GCP) Service-account", "\"type\": \"service_account\""),
        ("Twilio API Key", "SK[a-z0-9]{32}"),
        ("Password in URL", "[a-zA-Z]{3,10}://[^/\\s:@]{3,20}:[^/\\s:@]{3,20}@.{1,100}[\"'\\s]"),
    ];
    static REGEX_SET: Lazy<RegexSet> = Lazy::new(|| {
        RegexSet::new(RULES.iter().map(|&(_, regex)| regex)).expect("All regexes should be valid")
    });
```
The next step was to fetch the Dockerhub api for new public images, build and run them, and then scan the environment variables. 
This should repeat every few seconds. 

```rust
static DOCKER_API_URL: &str = "https://hub.docker.com/api/content/v1/products/search?page_size=100&q=%2B&source=community&type=image%2Cbundle&sort=updated_at";

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut sched = JobScheduler::new();

    #[cfg(feature = "signal")]
    sched.shutdown_on_ctrl_c();

    // cron every 30 seconds
    let four_s_job_async = Job::new_async("1/4 * * * * *", |_uuid, _l| Box::pin(async move {
             let secrets = docker::process(DOCKER_API_URL).await;
             docker::export_secrets(secrets.unwrap()).await.unwrap();
      })).unwrap();

    sched.add(four_s_job_async).unwrap();

    sched.set_shutdown_handler(Box::new(|| {
        Box::pin(async move {
            println!("Shut down done");
        })
    })).unwrap();

    sched.start().await.unwrap();
    
    Ok(())
}
```

I used a Rust library Tokio because it provides a way of running cron jobs with built in features that I found pleasing. 

### Results
After running the above for about three days I got a match!

`SLACK_TOKEN=xox...`

Obviously wont show the token here. 

I tried to reach out to the creater of the images, haven't heard back. 

Looking at some of the non matching data I found some other stuff, unsure
if any of it is actually sensetive. The rest is mostly gibberish. 

CASSANDRA_KEYSTORE_LOCATION=/bitnami/cassandra/secrets/keystore
spring.datasource.password=root123
