# 💡 WebDAV Integration: Storage Provider + Data Source Connector

## 🎯 Overview

I'd like to propose **WebDAV integration** for Open WebUI with **per-user credentials**, enabling two powerful use cases:

1. **📦 Storage Provider**: Use WebDAV (Nextcloud, ownCloud, Synology NAS) as storage backend for uploaded files
2. **🔗 Data Source Connector**: Browse and attach files directly from your cloud storage to Knowledge Bases/Chats

This would enable privacy-focused, self-hosted storage solutions while maintaining full compatibility with Open WebUI's existing architecture.

---

## 🌟 Why This Matters

### For Privacy-Conscious Users

Open WebUI attracts users who value **privacy and self-hosting**. Many of us already run Nextcloud, ownCloud, or Synology NAS for file storage. Currently, we need to:
- Either use local storage (limited scalability)
- Or configure S3/GCS/Azure (costs money, data leaves our infrastructure)

**With WebDAV support**, we could:
- ✅ Store all files in our existing self-hosted cloud
- ✅ Keep complete data sovereignty
- ✅ Zero additional infrastructure costs
- ✅ Use the same storage for Open WebUI and other apps

### For RAG/Knowledge Base Users

Currently, to add a document from Nextcloud to a Knowledge Base, I have to:
1. Download file from Nextcloud
2. Upload to Open WebUI
3. Now the file exists in two places
4. If I update the document in Nextcloud, the Knowledge Base is outdated

**With WebDAV Data Source**, we could:
- 📂 Browse Nextcloud folders directly in Open WebUI
- 🎯 Select files and add them to Knowledge Bases with one click
- 🔄 Keep files in sync (detect updates)
- 📎 Attach Nextcloud files directly to chat sessions

---

## 🔗 Related Discussions

This proposal relates to:
- **Issue #12724** - Nextcloud Documents Integration
- **Issue #5872** - Data Sources (broader external data integration)

While #5872 proposes general data source connectors, this focuses specifically on **WebDAV protocol** support. WebDAV is an open standard (RFC 4918) supported by multiple platforms, making it ideal for self-hosters.

---

## 💡 Proposed Solution

### Use Case 1: WebDAV as Storage Provider

**Current workflow:**
```
User uploads file → Stored locally or S3/GCS/Azure → Processed for RAG
```

**With WebDAV Storage:**
```
User uploads file → Stored in user's Nextcloud → Locally cached → Processed for RAG
```

**Key benefits:**
- Each user configures their own WebDAV credentials (privacy!)
- Files stored in user's personal cloud account
- Local caching for fast access
- Optional global credentials for shared storage scenarios

### Use Case 2: WebDAV Data Source Browser

**Proposed workflow:**
```
1. Configure WebDAV credentials in Settings
2. In Knowledge Base → "Add Files" → "Browse Cloud Storage"
3. Browse your Nextcloud directory tree
4. Select files → Added to Knowledge Base
5. System fetches from WebDAV → Processes → Indexes for RAG
```

**Key benefits:**
- No duplicate storage
- Real-time sync with cloud files
- Attach files from Nextcloud directly to chats
- Access entire document library without re-uploading

---

## 🏗️ Technical Architecture

### Per-User Credentials Approach

Instead of a shared WebDAV account (security risk!), I propose **per-user credentials**:

```json
{
  "user": {
    "settings": {
      "webdav": {
        "endpoint_url": "https://cloud.example.com/remote.php/dav/files/username/",
        "username": "myuser",
        "password": "encrypted-app-token",
        "root_path": "/openwebui",
        "verify_ssl": true
      }
    }
  }
}
```

**Why per-user?**
- 🔒 Security: Each user accesses only their own files
- 👥 Multi-tenancy: Perfect for shared Open WebUI instances
- 📊 Quota: Each user uses their own storage quota
- 🎯 Privacy: No shared credentials = no cross-user access

### Implementation Components

```
┌─────────────────────────────────────────────────────────────┐
│                        User Settings                         │
│              (Store WebDAV credentials)                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
       ┌───────────────┴───────────────┐
       │                               │
       ▼                               ▼
┌─────────────────┐         ┌──────────────────────┐
│ Storage Provider│         │ Data Source Connector│
│                 │         │                      │
│ - upload_file() │         │ - list_files()       │
│ - get_file()    │         │ - browse_directory() │
│ - delete_file() │         │ - fetch_for_rag()    │
└─────────────────┘         └──────────────────────┘
       │                               │
       └───────────────┬───────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  User's WebDAV  │
              │   (Nextcloud,   │
              │   ownCloud, NAS)│
              └─────────────────┘
```

### Backend Implementation

**1. Storage Provider** (`backend/open_webui/storage/webdav.py`)

Follows the existing `StorageProvider` ABC pattern (like S3, GCS, Azure):
- Extracts `user_id` from tags (already passed to storage methods)
- Loads user's WebDAV config from `user.settings.webdav`
- Uploads files to user's WebDAV account
- Local caching for performance
- No breaking changes to existing interface

**2. Data Source Connector** (`backend/open_webui/sources/webdav.py`)

New component for browsing WebDAV:
- List files in remote directories
- Fetch files for Knowledge Base processing
- Detect file changes for sync

**3. API Endpoints** (`backend/open_webui/routers/`)

```python
# Settings management
POST   /api/v1/users/user/settings/webdav       # Save credentials
GET    /api/v1/users/user/settings/webdav       # Get config (no password)
DELETE /api/v1/users/user/settings/webdav       # Delete credentials
POST   /api/v1/users/user/settings/webdav/test  # Test connection

# Data source browsing
GET    /api/v1/webdav/browse?path=/             # Browse directory
POST   /api/v1/webdav/files/add                 # Add files to Knowledge Base
```

### Frontend Implementation

**1. Settings UI** (`src/lib/components/settings/WebDAVSettings.svelte`)

Simple configuration form:
- WebDAV endpoint URL
- Username / Password (App Token recommended)
- Root path
- SSL verification toggle
- "Test Connection" button with visual feedback

**2. File Browser** (`src/lib/components/webdav/FileBrowser.svelte`)

For Phase 2:
- Tree view of WebDAV directory structure
- Multi-select for files/folders
- Search functionality
- "Add to Knowledge Base" action

### Security Features

- 🔐 **Encrypted Passwords**: WebDAV credentials encrypted using `cryptography.fernet`
- 🎫 **App Token Support**: UI recommends Nextcloud App Passwords (not main password)
- 🔒 **SSL by Default**: HTTPS with optional self-signed cert support
- 👤 **Isolation**: Each user only accesses their own WebDAV account

---

## 📋 Implementation Phases

### Phase 1: Storage Provider ⭐ (Core)

**What:**
- WebDAV storage backend for uploaded files
- User settings for WebDAV credentials
- Frontend settings UI
- Password encryption
- Tests + documentation

**Estimated effort:** ~18-25 hours

**Benefit:** Self-hosters can use Nextcloud instead of local/S3 storage

### Phase 2: Data Source Browser 🚀 (Extended)

**What:**
- Browse WebDAV directories
- Add files from WebDAV to Knowledge Bases
- File sync mechanism
- Chat attachment from cloud
- Frontend file browser UI

**Estimated effort:** ~20-30 hours

**Benefit:** Full RAG integration with existing cloud documents

### Phase 3: Advanced Features 🎁 (Optional)

**What:**
- Multi-account support (multiple WebDAV connections)
- Selective folder sync
- Webhook support for real-time updates
- Quota display
- Admin override capabilities

---

## 🎬 Example Use Cases

### Example 1: Privacy-Focused Self-Hoster

**Sarah** runs Open WebUI on her homelab and uses Nextcloud for all her files.

**Today:**
```
1. Upload PDF to Open WebUI
2. File stored locally on homelab
3. File also in Nextcloud (duplicate)
4. Managing two copies manually
```

**With WebDAV:**
```
1. Configure Nextcloud credentials once
2. Upload PDF to Open WebUI
3. Automatically stored in Nextcloud
4. Accessible from both Open WebUI and Nextcloud mobile app
5. Single source of truth
```

### Example 2: Researcher with Large Document Library

**Dr. Martinez** has 10GB of research papers in Nextcloud folders organized by topic.

**Today:**
```
1. Download papers from Nextcloud
2. Upload to Open WebUI Knowledge Base
3. Now has two copies of 10GB
4. When updating a paper, must re-upload
```

**With WebDAV Data Source:**
```
1. Connect Nextcloud to Open WebUI
2. Browse folders directly in Knowledge Base interface
3. Select "Machine Learning Papers" folder
4. Add to Knowledge Base with one click
5. Papers stay in Nextcloud, instant RAG access
```

### Example 3: Enterprise Team

**Company** uses Open WebUI with shared Synology NAS for compliance.

**Today:**
```
- Files scattered between Open WebUI local storage and NAS
- Compliance team can't audit Open WebUI files on NAS
- Each user needs local disk space
```

**With WebDAV:**
```
- Each employee configures their NAS account
- All files stored on company NAS (compliance ✓)
- IT can audit all files in one place
- No local storage needed
```

---

## ❓ Questions for the Community

I'd love to hear your thoughts on:

### 1. **Scope**: Which phase is most valuable to you?
   - [ ] Phase 1 only (Storage Provider)
   - [ ] Both Phase 1 + 2 (Storage + Data Source)
   - [ ] All three phases

### 2. **Use Case**: What's your primary need?
   - [ ] Replace S3/GCS with self-hosted storage
   - [ ] Access existing Nextcloud documents for RAG
   - [ ] Both equally important
   - [ ] Other (please comment)

### 3. **Platform**: What WebDAV server do you use?
   - [ ] Nextcloud
   - [ ] ownCloud
   - [ ] Synology NAS
   - [ ] Other WebDAV server (please specify)

### 4. **Deployment**: How do you run Open WebUI?
   - [ ] Self-hosted (Docker)
   - [ ] Self-hosted (non-Docker)
   - [ ] Shared instance (multiple users)
   - [ ] Cloud deployment

### 5. **UI Preference**: Where should WebDAV settings appear?
   - [ ] Settings → Storage → WebDAV
   - [ ] Settings → Account → External Storage
   - [ ] Dedicated "Connections" section
   - [ ] Don't care / other suggestion

---

## 🚀 Next Steps

If this proposal gets positive feedback from the community and maintainers, I'm willing to:

1. ✅ Implement Phase 1 (Storage Provider)
2. ✅ Submit PR with tests + documentation
3. ✅ Collaborate on Phase 2 if there's interest
4. ✅ Provide setup guides for Nextcloud/ownCloud/Synology

**My background:**
- Python backend development (FastAPI, SQLAlchemy)
- WebDAV protocol integration experience
- Svelte frontend development
- Active Open WebUI user and self-hoster

---

## 🔗 Technical References

- **WebDAV Spec**: [RFC 4918](https://datatracker.ietf.org/doc/html/rfc4918)
- **Nextcloud WebDAV API**: [Documentation](https://docs.nextcloud.com/server/latest/developer_manual/client_apis/WebDAV/index.html)
- **Proposed Library**: [webdavclient3](https://github.com/ezhov-evgeny/webdav-client-python-3)
- **Alternative**: [NC-Py-API](https://github.com/cloud-py-api/nc_py_api) (Nextcloud-specific)

---

## 💬 Let's Discuss!

**What do you think?**
- Does this solve a problem you have?
- Would you use this feature?
- Any concerns or suggestions?
- Which phase is most important to you?

I'm looking forward to your feedback and ideas! 🎉

---

**Related Issues:**
- #12724 - Nextcloud Documents Integration
- #5872 - Data Sources feature request
