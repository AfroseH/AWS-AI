# AWS Lab – Amazon Transcribe Speech-to-Text with Transcript Extraction

## Overview

Amazon Transcribe is an automatic speech recognition (ASR) service that uses machine learning to convert speech into text.

In this lab, we upload an audio file to Amazon S3, which triggers an AWS Lambda function that starts an Amazon Transcribe job. Transcribe stores the raw transcript as a JSON file in S3, which triggers a second Lambda function. That function extracts the readable transcript text and saves it as a clean `.txt` file in S3.

---

## Architecture

### Components

- Amazon S3
- AWS Lambda
- Amazon Transcribe
- AWS IAM

![Architecture](https://github.com/AfroseH/AWS-AI/blob/main/architecture.png)

---

## Step 1: Create S3 Bucket

An Amazon S3 bucket named `automate-transcribe-job-lab05` was used to store the input audio file, the raw transcript JSON (`transcripts/`), and the clean text output (`clean-text/`).

![S3 Bucket](https://github.com/AfroseH/AWS-AI/blob/main/S3.png)

---

## Step 2: Create IAM Role for the Transcribe Lambda

An IAM execution role was created and attached to the Lambda function that starts the transcription job.

![Transcribe Role](https://github.com/AfroseH/AWS-AI/blob/main/transcribe-role.png)

---

## Step 3: Create Lambda Function 1 (Start Transcription Job)

An AWS Lambda function was created using Python (boto3).

The Lambda function:

- Receives the S3 event when an audio file is uploaded.
- Reads the bucket name and object key from the event.
- Detects the audio format from the file extension (for example `mp3`, `wav`).
- Starts an Amazon Transcribe job with a unique name (`transcription-<timestamp>`), using language `en-US`.
- Saves the transcript JSON to `transcripts/<job-name>.json` in the output bucket.

```python
import boto3
import urllib.parse
import os
import time

transcribe = boto3.client("transcribe")

OUTPUT_BUCKET = "automate-transcribe-job-lab05"


def lambda_handler(event, context):

    # Get S3 bucket name
    bucket = event["Records"][0]["s3"]["bucket"]["name"]

    # Get uploaded file name
    key = urllib.parse.unquote_plus(
        event["Records"][0]["s3"]["object"]["key"]
    )

    print(f"Audio file uploaded: s3://{bucket}/{key}")

    # Get file name
    file_name = os.path.basename(key)

    # Create unique Transcribe job name
    job_name = f"transcription-{int(time.time())}"

    # Get audio format
    media_format = file_name.split(".")[-1].lower()

    # Start Transcribe job
    transcribe.start_transcription_job(
        TranscriptionJobName=job_name,

        Media={
            "MediaFileUri": f"s3://{bucket}/{key}"
        },

        MediaFormat=media_format,

        LanguageCode="en-US",

        OutputBucketName=OUTPUT_BUCKET,

        OutputKey=f"transcripts/{job_name}.json"
    )

    print(f"Transcription job started: {job_name}")

    return {
        "statusCode": 200,
        "jobName": job_name
    }
```

![Transcribe Function](https://github.com/AfroseH/AWS-AI/blob/main/transcribe-function.png)

---

## Step 4: Add S3 Trigger and Verify Transcription Job

An S3 trigger was added to the Lambda function so that uploading an audio file starts the process automatically.

The transcription job was created in Amazon Transcribe and completed successfully.

![Transcription Job](https://github.com/AfroseH/AWS-AI/blob/main/transcriptionjob.png)

---

## Step 5: View Raw Transcript Output

Amazon Transcribe generated the transcript as a JSON file in the `transcripts/` folder. The JSON contains the full transcript along with metadata such as word-level timestamps and confidence scores.

![Transcript Output](https://github.com/AfroseH/AWS-AI/blob/main/transcript-op.png)

---

## Step 6: Create IAM Role for the Clean Transcript Lambda

A separate IAM execution role was created for the Lambda function that processes the transcript JSON.

### Permissions

- Amazon S3 `GetObject` and `PutObject`
- CloudWatch Logs

![Clean Transcript Role](https://github.com/AfroseH/AWS-AI/blob/main/cleantranscribe-role.png)

---

## Step 7: Create Lambda Function 2 (Extract Clean Transcript)

A second AWS Lambda function was created using Python (boto3) and triggered by the transcript JSON file created in S3.

The Lambda function:

- Receives the S3 event when a transcript JSON file is created.
- Reads the JSON file from S3.
- Extracts the transcript text from `results.transcripts[0].transcript`.
- Saves the clean text as a `.txt` file in the `clean-text/` folder.

```python
import boto3
import json
import urllib.parse

s3 = boto3.client("s3")

OUTPUT_BUCKET = "automate-transcribe-job-lab05"


def lambda_handler(event, context):

    print("S3 event received:")
    print(json.dumps(event))

    # Get bucket name
    bucket = event["Records"][0]["s3"]["bucket"]["name"]

    # Get JSON file key
    key = urllib.parse.unquote_plus(
        event["Records"][0]["s3"]["object"]["key"]
    )

    print(f"Input JSON: s3://{bucket}/{key}")

    # Read Transcribe JSON from S3
    response = s3.get_object(
        Bucket=bucket,
        Key=key
    )

    transcript_data = json.loads(
        response["Body"].read().decode("utf-8")
    )

    # Extract readable transcript
    clean_text = transcript_data["results"]["transcripts"][0]["transcript"]

    print("Clean text:")
    print(clean_text)

    # Create output filename
    file_name = key.split("/")[-1]
    base_name = file_name.rsplit(".", 1)[0]

    output_key = f"clean-text/{base_name}.txt"

    # Save clean text to S3
    s3.put_object(
        Bucket=OUTPUT_BUCKET,
        Key=output_key,
        Body=clean_text.encode("utf-8"),
        ContentType="text/plain"
    )

    print(f"Clean text saved to: s3://{OUTPUT_BUCKET}/{output_key}")

    return {
        "statusCode": 200,
        "message": "Clean transcript created successfully",
        "output": f"s3://{OUTPUT_BUCKET}/{output_key}"
    }
```

![Clean Transcript Function](https://github.com/AfroseH/AWS-AI/blob/main/cleantranscript-function.png)

---

## Step 8: Add S3 Trigger for the Clean Transcript Lambda

An S3 trigger was added to the second Lambda function.

- Event Type: **All object create events**
- Prefix: `transcripts/`
- Suffix: `.json`

---

## Step 9: View Clean Text Output

The clean transcript was stored in the `clean-text/` folder as a `.txt` file, which is much easier to read than the raw JSON.

![Clean Text Output](https://github.com/AfroseH/AWS-AI/blob/main/cleantxt-op.png)

---

## Step 10: Final Output

The final output shows the uploaded audio converted into readable text.

![Final Output](https://github.com/AfroseH/AWS-AI/blob/main/final-op.png)

---

## Notes

- Transcribe jobs are asynchronous. The JSON file appears in S3 only after the job status is `COMPLETED`.
- Each Lambda writes to a different prefix (`transcripts/` and `clean-text/`) and the second trigger is filtered to `transcripts/` and `.json`, so the functions do not trigger themselves in a loop.
- The first Lambda detects the audio format from the file extension, so use a supported format such as `mp3`, `wav`, `mp4`, or `flac`.
- Delete the Lambda functions, S3 files, and transcription jobs after the lab to avoid unexpected charges.

---

## Conclusion

In this lab, we:

- Created an Amazon S3 bucket for audio, transcript, and text files.
- Created IAM roles for both Lambda functions.
- Created a Lambda function to start an Amazon Transcribe job from an S3 upload.
- Generated a raw transcript in JSON format.
- Created a second Lambda function to extract the transcript text.
- Stored the clean text output in Amazon S3.

This lab demonstrates an event-driven speech-to-text workflow using Amazon S3, AWS Lambda, and Amazon Transcribe.
