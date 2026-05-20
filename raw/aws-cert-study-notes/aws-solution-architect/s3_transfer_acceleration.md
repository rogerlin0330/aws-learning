# S3 Transfer Acceleration

This content focuses on **performance optimization techniques for Amazon S3**, particularly in the context of a distributed organization scenario, "Animals for Life," which involves a head office in Brisbane, remote offices, and remote workers uploading large datasets over often unreliable internet connections.

![[s3-transfer-acceleration-infographic.png]]

## Key Insights on S3 Upload Performance

- **Default S3 Upload Behavior**:
  - Uploads use a **single stream (single put object API call)** to transfer data as a single blob.
  - This is **simple but unreliable** for large files because if the stream fails, the entire upload must be restarted.
  - Example risk: a failure at 4.5 GB of a 5 GB upload results in losing the entire 4.5 GB transmission and wasting time.

- **Limitations of Single Stream Transfers**:
  - Single streams often **underperform relative to available bandwidth**, especially over long distances or unreliable networks.
  - Transfers are limited by the **slowest link between sender and receiver**, and single streams may not even reach that threshold.
  - AWS limits single put uploads to a maximum size of **5 GB**.
  - Single put uploads are **not recommended for files near this size due to unreliability**.
	- **Single PUT (up to 5 GB)**
		  - `PutObject` — uploads the entire object in one request.
    - **Multipart upload (for files > 5 GB, up to ~5 TB)**
	    - It's a sequence of calls, not a single API:
		    1. `CreateMultipartUpload` — initiates the upload and returns an `UploadId`.
		    2. `UploadPart` — called once per part (5 MiB–5 GiB each, max 10,000 parts), referencing the `UploadId`. Returns an `ETag` per part.
		    3. `CompleteMultipartUpload` — assembles the parts into the final object using the list of part numbers and ETags.
		    4. `AbortMultipartUpload` — cancels and cleans up if something goes wrong (important — incomplete uploads still incur storage charges until aborted or completed).
		- Helper APIs you'll likely also touch:
			- `ListParts` — list parts already uploaded for a given `UploadId`.
			- `ListMultipartUploads` — list in-progress multipart uploads in a bucket (useful for finding orphaned uploads to abort, often combined with an S3 Lifecycle rule like `AbortIncompleteMultipartUpload` to auto-clean after N days).
	- **Practical note:** if you're using an AWS SDK (boto3, Java v2, etc.) or the CLI (`aws s3 cp`), the high-level transfer manager handles all of this for you — it automatically picks single PUT vs. multipart based on file size, parallelizes part uploads, and retries failed parts. You generally only call the low-level multipart APIs directly when you need fine-grained control (resumable uploads across processes, custom parallelism, server-side composition, etc.).

## Multi-Part Upload: The Recommended Solution

- **Multi-part upload** breaks data into multiple parts to improve performance and reliability.
- **Key parameters**:
  - Minimum file size to use multi-part upload: **100 MB**.
  - Maximum number of parts: **10,000**.
  - Each part size ranges between **5 MB and 5 GB**; the last part can be smaller than 5 MB.
- **Advantages**:
  - Parts upload independently, so if one part fails, only that part needs to be retried.
  - Enables **parallel uploads**, significantly increasing overall transfer rates.
  - Helps better utilize available bandwidth by mitigating single-stream network inefficiencies.
- **Recommendation**: Use multi-part upload for all files larger than 100 MB. Most AWS tools enable this automatically at this threshold.

## S3 Transfer Acceleration

- Addresses global upload performance issues caused by **indirect and inefficient routing over the public internet**.
- Scenario: Distributed teams (Australia, South America, US West Coast) uploading data to an S3 bucket located in London.
- Public internet routing is often **non-optimal**, sometimes routing data via unexpectedly long paths, causing slowdowns.
- **How Transfer Acceleration works**:
  - Utilizes AWS **edge locations** globally.
  - Uploads first go to the **nearest AWS edge location**, which is geographically closer and involves fewer network hops.
  - From the edge location, data transits over the **AWS global private network**, a purpose-built high-speed network connecting AWS regions.
- Benefits:
  - **Higher throughput and reliability** for uploads, especially over long distances.
  - Performance gains increase with the distance between the upload source and the S3 bucket.
  - More significant improvements are seen when initial connections are poor.
- **Bucket requirements**:
  - Bucket must be enabled for transfer acceleration (off by default).
  - Bucket name must be **DNS-compatible and cannot contain periods**.

## Conceptual Comparison: Public Internet vs. AWS Network

| Feature              | Public Internet                      | AWS Global Network                       |
| -------------------- | ------------------------------------ | ---------------------------------------- |
| Routing              | Multiple hops, indirect paths        | Direct, optimized routes between regions |
| Performance          | Variable, often slower               | Consistent, high-speed, low latency      |
| Design goals         | Flexibility and resilience           | Speed and reliability                    |
| Control over routing | None (ISPs and routers decide paths) | Fully controlled by AWS                  |

## Additional Notes

- The content mentions a **demonstration lesson** to show how to enable transfer acceleration and measure performance benefits using AWS tools.
- Emphasizes the critical nature of timely data uploads for the organization’s operation and the impact on animal welfare.

## Core Conclusions

- **Multi-part upload is essential for reliable, fast uploads of large files to S3, particularly over unreliable or slow networks.**
- **S3 Transfer Acceleration significantly improves upload speed and reliability for geographically dispersed teams by leveraging AWS edge locations and private global network infrastructure.**
- Single stream uploads are only suitable for small files (<100 MB) or non-critical transfers due to their inherent reliability and speed limitations.
- Understanding and implementing these features is crucial for solutions architects working with distributed organizations requiring efficient cloud storage access.

## Keywords

- Amazon S3
- Single put upload
- Multi-part upload
- Upload performance
- Upload reliability
- Transfer Acceleration
- AWS edge locations
- Distributed teams
- Network routing
- AWS global network
