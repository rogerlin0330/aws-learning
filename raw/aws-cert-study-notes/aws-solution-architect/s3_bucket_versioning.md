# S3 Object Versioning and MFA Delete

**Object Versioning** and **MFA Delete** are two critical features of Amazon S3, frequently tested in certification exams and vital in real-world projects involving data protection and management.

---

## Object Versioning

- **Controlled at the bucket level**, S3 versioning begins in a **disabled state**.
- Versioning states and transitions:
  - Starts as **Disabled**.
  - Can be **Enabled** from Disabled.
  - Once **Enabled**, versioning **cannot be disabled**, only **Suspended**.
  - From **Suspended**, versioning can be **re-enabled**.
  ![[s3-bucket-versioning-state.png]]

- **Key exam takeaway:**  
  - **Enabled → Disabled is impossible** (cannot revert to Disabled once enabled).
  - You can only toggle between **Enabled** and **Suspended** states after enabling.

- Without versioning, objects are identified solely by their **unique key (object name)**, and any modification overwrites the existing object.
- With versioning enabled:
  - Multiple versions of the same object can be stored.
    ![[s3-object-versioning.png]]
  - Each version has a unique **version ID** assigned by S3.
  - The **current version** is the most recent version of an object.
  - Accessing an object without specifying a version ID returns the **current version** by default.
  - You can explicitly request any previous version by its version ID.

- **Deletion behavior with versioning:**
  - Deleting an object without specifying a version ID adds a **delete marker** instead of removing data.
  - ![[s3-object-delete-marker.png.png]]
  - The delete marker is a special version that hides all prior versions, making the object appear deleted.
  - Removing the delete marker **restores** the object’s visibility.
  - To permanently delete a specific version, you must specify its version ID.
  - If deleting the current version, the **next latest version becomes current**.
  - If the last version of an object is deleted, then the object is permanently deleted.
    ![[s3-object-deletion-if-versioning-enabled.png]]

- **Storage implications:**
  - All versions consume storage space.
  - Example: Five versions of a 5 GB object consume $5 \times 5 \text{ GB} = 25 \text{ GB}$.
  - Suspending versioning does **not** delete existing versions or reduce storage usage.
  - The only way to remove all versions and reduce costs is to delete the bucket and recreate it without versioning.

---

## MFA Delete

- An additional security feature configurable only on buckets with versioning enabled.
- Requires **Multi-Factor Authentication (MFA)** to:
  - Change versioning state (from Enabled to Suspended or vice versa).
  - Permanently delete specific object versions.
- MFA usage in API calls:
  - Must provide the **MFA device serial number** and the **current MFA code**.
  - These credentials are included in API requests for sensitive operations.

---

## Timeline of Versioning States

| Bucket Versioning State | Allowed Transitions     | Notes                                  |
| ----------------------- | ----------------------- | -------------------------------------- |
| Disabled                | Can enable → Enabled    | Initial state; no version IDs assigned |
| Enabled                 | Can suspend → Suspended | Cannot disable once enabled            |
| Suspended               | Can re-enable → Enabled | Existing versions retained             |

---

## Key Insights

- **Versioning cannot be disabled once enabled**; only suspended.
- **Delete markers** enable soft deletion, hiding but not removing data.
- Permanent deletion requires specifying the version ID.
- **MFA Delete adds an extra layer of protection** for critical versioning changes and deletions.
- Versioning can lead to significant **storage cost increases** due to multiple retained versions.
- Understanding versioning states and MFA Delete processes is essential for both operational management and exam success.

---

This summary encapsulates the essential concepts and operational mechanics of S3 object versioning and MFA Delete as presented in the lesson, providing a clear, reliable reference for study or practical application.