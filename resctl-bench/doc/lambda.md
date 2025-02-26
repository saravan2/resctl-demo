Lambda function
===============

The lambda subcommand is used to provide automated ingestion of benchmark results and filing them as github issues. The repository used for collecting the files is `iocost-benchmarks`:

  https://github.com/iocost-benchmark/iocost-benchmarks/issues

Building
========

To build resctl-bench for use as the lambda you need to take into account the architecture you want to run it on and the fact that the base system is Amazon Linux 2. This is important because if you end up linking to libraries dynamically the binary will not run.

The safest way to do it is to use the musl target, which links statically, so that glibc version is not a concern.

Start by adding the target to your rust setup:

```
$ rustup target add x86_64-unknown-linux-musl
```

Then do a release build with the lambda feature enabled and the musl target. The example also uses a separate target directory to keep this build separate from the day to day development ones:

```
$ cargo build --release --bin resctl-bench --features lambda --target-dir aws-target --target x86_64-unknown-linux-musl
```

AWS setup
=========

Before the function is deployed, there are a few configurations and permissions to set up. In particular, the function uses the AWS System Manager Parameter Store to obtain the token used for logging in to github, and an S3 bucket to store the results that it receives.

First, we need an IAM role that the function will run as. It should let AWS lambda use AssumeRole on it, which means this `Trust relationship`:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Then it needs permission to be able to, first of all, run the lambda function and save logs. That can be achieved by giving it the `AWSLambdaBasicExecutionRole` managed policy. In addition, it needs to be able to read the Parameter Store, so we can give it the `AmazonSSMReadOnlyAccess` managed policy.

For S3 it is best to give the function only exactly what it needs. So we create an `iocost-submit` bucket and give access to write to it to the lambda function, as well as reading and listing, which are required to check if the file already exists:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "IoCostSubmitRW",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::iocost-submit/*",
                "arn:aws:s3:::iocost-submit"
            ]
        }
    ]
}
```

We still need to configure our S3 bucket to allow public read only access, so that the link we post to github can be used to download the file. First of all, on bucket settings we need to disable 'Block all public access', then we can add the following policy to open up the read-only access:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::iocost-submit/*"
        }
    ]
}
```

Deploying
=========

AWS Lambda expects a `bootstrap` binary contained on a zip file. resctl-bench will check if that is its name and re-exec itself with the appropriate arguments, since AWS does not pass any arguments. First of all, build the zip file:

```
$ cp aws-target/x86_64-unknown-linux-musl/release/resctl-bench ./bootstrap
$ zip resctl-bench-lambda.zip bootstrap && rm -f bootstrap
```

Now we can deploy it to AWS:

```
$ aws lambda create-function --function-name "iocost-submit" \
    --handler bundle \
    --architectures "<ARCHITECTURE>" \
    --memory-size 256 \
    --timeout 15 \
    --role "<IAM_ROLE_ARN>" \
    --zip-file fileb://resctl-bench-lambda.zip \
    --runtime provided.al2 \
    --environment Variables={RUST_BACKTRACE=1} \
    --tracing-config Mode=Active
```

If you are just updating the code, then this is sufficient:

```
$ aws lambda update-function-code --function-name "iocost-submit" \
    --zip-file fileb://resctl-bench-lambda.zip
```
