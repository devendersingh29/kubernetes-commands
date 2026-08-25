# kubernetes-commands


# CBT Backfill Script — Command Reference

Branch: `CBT-status-bug-august-kuldeep`
Service: `pms-assessment-api`
Script: `backfill_cbt_status.py`
Environment: Dev

This document lists every command used to locate, verify, and run the CBT backfill script in the dev environment, along with what each command does.

---

## 1. Cluster / Pod Discovery

### List pods in the database namespace
```bash
k get pods -n pms-database
```
Shows all running pods (Mongo primary/secondary replicas, backup jobs, mongodump utility pods, etc.) in the `pms-database` namespace.

### List services in the database namespace
```bash
k get svc -n pms-database -o wide
```
Shows all Kubernetes Services and which pods they route to (via the `SELECTOR` column). Used to find the NodePort services exposing MongoDB outside the cluster.

### List cluster nodes with IPs
```bash
k get nodes -o wide
```
Shows every node in the cluster along with its `INTERNAL-IP`. Needed because NodePort services are exposed on **every node's IP**, not the internal ClusterIP.

### Find the app pod for a given service
```bash
k get pods -A | grep pms-assessment-api
```
Searches across all namespaces for pods belonging to the `pms-assessment-api` service — used to locate the pod in `pms-dev` where the actual (correct) script and config already lived.

---

## 2. MongoDB Access & Verification

### Get the replica set name
```bash
k exec -it mongo-primary-0 -n pms-database -- mongosh -u user -p '<password>' --authenticationDatabase admin --eval "rs.status().set"
```
Connects into the `mongo-primary-0` pod and runs a `mongosh` command to print the replica set name (`rs0`), required for building a proper connection URI.

### List all databases
```bash
k exec -it mongo-primary-0 -n pms-database -- mongosh -u root -p '<password>' --authenticationDatabase admin --eval "show dbs"
```
Confirms which databases exist on the cluster — used to verify `wtt_pms_dev` and `wtt_assessment_dev` exist and that no obvious production DB was present.

### List collections in a specific database
```bash
k exec -it mongo-primary-0 -n pms-database -- mongosh -u root -p '<password>' --authenticationDatabase admin --eval "db.getSiblingDB('wtt_assessment_dev').getCollectionNames()"
```
Lists every collection inside `wtt_assessment_dev`. Used to confirm whether `candidate_assessment_schedules` (from `config.py`) actually existed as a collection name.

### Fetch a sample document from a collection
```bash
k exec -it mongo-primary-0 -n pms-database -- mongosh -u root -p '<password>' --authenticationDatabase admin --eval "db.getSiblingDB('wtt_assessment_dev').candidate_assessments.findOne()"
```
Pulls one document from `candidate_assessments` to inspect its actual field names/schema — used to check for `is_evaluated`, `overall_score`, `scheduleId`, etc.

### Find a document matching a specific field condition
```bash
k exec -it mongo-primary-0 -n pms-database -- mongosh -u root -p '<password>' --authenticationDatabase admin --eval "db.getSiblingDB('wtt_assessment_dev').candidate_assessments.findOne({is_evaluated: {\$exists: true}})"
```
Searches specifically for a document where the `is_evaluated` field exists, to confirm whether that field is present in at least some records (since the first sampled document didn't have it).

### Inspect the other side of the relationship (`interviewschedules`)
```bash
k exec -it mongo-primary-0 -n pms-database -- mongosh -u root -p '<password>' --authenticationDatabase admin --eval "db.getSiblingDB('wtt_pms_dev').interviewschedules.findOne()"
```
Pulled a sample document from `interviewschedules` to check whether its `_id` matches the `scheduleId` format used in `candidate_assessments` — revealed a schema mismatch (UUID string vs. ObjectId, and no `scheduleId` field present at all).

---

## 3. Local Environment Setup (walkingtree machine)

### Take a backup before running any migration
```bash
mongodump \
  --host localhost \
  --port 27017 \
  --username user \
  --password <password> \
  --authenticationDatabase admin \
  --db wtt_pms_dev \
  --out ./mongodb-backup

mongodump \
  --host localhost \
  --port 27017 \
  --username user \
  --password <password> \
  --authenticationDatabase admin \
  --db wtt_assessment_dev \
  --out ./mongodb-backup
```
Creates a local backup of both dev databases before running any write operation, in case a rollback is needed.

### Install Python package manager (if missing)
```bash
sudo apt update
sudo apt install python3-pip -y
```
Installs `pip3`, required to install the script's Python dependencies.

### Install script dependencies
```bash
pip3 install motor pymongo
```
Installs `motor` (async MongoDB driver) and `pymongo` (sync MongoDB driver/utility classes like `UpdateOne`, `ObjectId`), both required by `backfill_cbt_status.py`.

---

## 4. Connecting to MongoDB from Outside the Cluster

### Build the connection URI (NodePort-based)
```
mongodb://<username>:<password>@<any-node-internal-ip>:32217/?replicaSet=rs0&authSource=admin
```
- `32217` is the NodePort exposing `mongo-primary-0` (port `27017` internally) on every cluster node.
- Any `Ready` worker node's `INTERNAL-IP` can be used — NodePort forwards traffic to the correct pod regardless of which node IP is used.
- Secondary NodePorts (`32218`, `32219`) exist but aren't required for a single-seed connection.

This value goes into `config.py` as the `URI` variable — it is **not** run directly in the terminal.

---

## 5. Working Inside the `pms-assessment-api` Pod

### Exec into the pod with a shell
```bash
k exec -it pms-assessment-api-86dc6f99c5-dmkqm -n pms-dev -- bash
```
Opens an interactive shell inside the running application pod, where the real deployed script and config already exist (at `/app`).

### Check Python version / dependencies inside a pod
```bash
k exec -it <pod-name> -n <namespace> -- python3 --version
k exec -it <pod-name> -n <namespace> -- pip3 list 2>/dev/null | grep -i -E "motor|pymongo"
```
Confirms whether a given pod has Python and the required libraries installed before attempting to run the script there.

### Copy a local file into a pod
```bash
k cp backfill_cbt_status.py pms-dev/pms-assessment-api-86dc6f99c5-dmkqm:/app/backfill_cbt_status.py
```
Copies a file from the local machine into a specific path inside a running pod.

### Compare two versions of a file
```bash
diff /app/backfill_cbt_status.py /tmp/backfill_cbt_status.py
```
Shows line-by-line differences between two files — used to confirm whether the deployed script and a locally copied version were functionally identical (in this case, only whitespace differed).

### Read environment variables inside the pod
```bash
echo $MONGODB_CANDIDATE_ASSESSMENT_SCHEDULES
echo $MONGODB_DATABASE_NAME
echo $MONGODB_ASSESSMENT_DB_NAME
echo $MONGODB_URL
```
`/app/config.py` inside this pod loads all values from environment variables (via `os.environ`) rather than hardcoded strings. These commands reveal the actual, real config values the running service uses — the authoritative source of truth, instead of guessing.

### Create/overwrite a file directly inside the pod (no editor needed)
```bash
cat > backfill_cbt_status.py << 'PYEOF'
<file contents>
PYEOF
```
Used because the pod's container image doesn't include `nano` or `vi`. This heredoc syntax writes multi-line file content directly from the shell without needing a text editor.

### Verify file contents after writing
```bash
cat backfill_cbt_status.py
```
Prints the full file contents to confirm the correct version was saved.

### Clean up temporary files
```bash
rm /tmp/config.py /tmp/backfill_cbt_status.py
rm -rf /tmp/__pycache__
```
Removes locally-copied files from `/tmp` inside the pod once no longer needed — important because the temporary `config.py` contained a plaintext database password.

---

## 6. Running the Script

### Run the backfill script
```bash
cd /app
python3 backfill_cbt_status.py
```
Executes the migration script using the pod's real environment-based config. The script:
1. Connects to MongoDB.
2. Queries `hiringworkflows` for candidates with a CBT stage stuck at status `"N"` with a past `endDate`.
3. Cross-checks each one against `candidate_assessments` to see if they actually completed the assessment (`is_evaluated == True`).
4. For matches, calculates a pass/fail recommendation from `overall_score` and prepares updates.
5. Executes `bulk_write` on `hiringworkflows` (and attempts `interviewschedules`, though this currently fails silently — see Known Issue below).
6. Prints a summary of how many documents were found and modified.

**Result of first dev run:** `Found 0 potentially stuck CBT workflows.` — no candidates in dev currently match the stuck-CBT criteria, so no updates occurred.

---

## 7. Known Issue (Not Yet Fixed — Flagged to Developer)

The `interviewschedules` update logic assumes `candidate_assessments.scheduleId` (a UUID string) can be converted to an `ObjectId` and matched against `interviewschedules._id` (a standard ObjectId). In practice:

- `interviewschedules` has **no `scheduleId` field at all**.
- The UUID string is not a valid `ObjectId` format, so the conversion always throws.
- The exception is silently caught by a bare `except: pass`, so `interviewschedules` is **never updated**, with no visible error.

`hiringworkflows` updates are unaffected by this bug and work correctly.

**Suggested fix (pending developer confirmation):** match `interviewschedules` using `jobId` + `resumeId` (fields shared with `hiringworkflows`) instead of `scheduleId`/`_id`.
