
# CMS UML Diagrams

This document collects all UML diagrams for the CMS architecture used in the laboratory work.

---

# Class Diagram

```plantuml
@startuml
hide circle
skinparam classAttributeIconSize 0

abstract class BaseEntity {
  -id: UUID
}

abstract class AuditedEntity {
  -createdAt: timestamptz
  -updatedAt: timestamptz
}

AuditedEntity --|> BaseEntity

class User {
  -email: varchar(255)
  -passwordHash: varchar(255)
  -isVerified: boolean
  -lastLoginAt: timestamptz

  +verifyEmail(now: timestamptz): void
  +recordLogin(now: timestamptz): void
}
User --|> AuditedEntity

class Site {
  -domain: varchar(255) <<unique>>
  -title: varchar(255)
  -description: text
  -deletedAt: timestamptz <<nullable>>

  +updateTitle(title: varchar(255), now: timestamptz): void
  +softDelete(now: timestamptz): void
}
Site --|> AuditedEntity

class SiteUser {
  -siteId: UUID
  -userId: UUID
  -role: SiteRole
  -createdAt: timestamptz

  +changeRole(newRole: SiteRole): void
}
SiteUser --|> BaseEntity

class Role {
  -name: string
}
class Permission {
  -code: string
}

abstract class ContentItem {
  -siteId: UUID
}
ContentItem --|> AuditedEntity

class Post {
  -authorId: UUID
  -title: varchar(255)
  -slug: varchar(255) <<unique per Site>>
  -content: text
  -status: PostStatus
  -publishedAt: timestamptz <<nullable>>
  -archivedAt: timestamptz <<nullable>>

  +edit(title: varchar(255), content: text, now: timestamptz): void
  +publish(now: timestamptz): void
  +archive(now: timestamptz): void
}
Post --|> ContentItem

class Comment {
  -postId: UUID
  -authorUserId: UUID <<nullable>>
  -authorName: varchar(255)
  -authorEmail: varchar(255)
  -text: text
  -status: CommentStatus
  -approvedAt: timestamptz <<nullable>>
  -rejectedAt: timestamptz <<nullable>>

  +approve(now: timestamptz): void
  +reject(now: timestamptz): void
}
Comment --|> AuditedEntity

class Media {
  -uploaderId: UUID
  -filePath: varchar(512)
  -fileType: varchar(100)
  -sizeBytes: bigint
  -isPrivate: boolean

  +markPrivate(): void
  +markPublic(): void
}
Media --|> ContentItem

class ContentLibrary {
  -name: varchar(255)
  -createdAt: timestamptz
}
ContentLibrary --|> BaseEntity

class LibraryAsset {
  -name: varchar(255)
  -createdAt: timestamptz
}
LibraryAsset --|> BaseEntity

class AssetVersion {
  -assetId: UUID
  -versionNumber: int
  -filePath: varchar(512)
  -createdAt: timestamptz
  -createdBy: UUID
  -isActive: boolean

  +activate(now: timestamptz): void
  +deactivate(now: timestamptz): void
}
AssetVersion --|> BaseEntity

interface SpamService {
  +checkSpam(text: text): boolean
}
class AkismetSpamService
class SimpleSpamService
AkismetSpamService ..|> SpamService
SimpleSpamService ..|> SpamService

interface PluginHook {
  +beforePublish(postId: UUID): void
  +afterPublish(postId: UUID): void
}
class SeoPluginHook
class AnalyticsPluginHook
SeoPluginHook ..|> PluginHook
AnalyticsPluginHook ..|> PluginHook

interface StorageService {
  +storeMedia(filePath: varchar(512)): void
  +storeAssetVersion(filePath: varchar(512)): void
}
class SiteMediaStorage
class LibraryAssetStorage
SiteMediaStorage ..|> StorageService
LibraryAssetStorage ..|> StorageService

User "1" -- "*" SiteUser
Site "1" -- "*" SiteUser
SiteUser "*" -- "1" Role
Role "1" -- "*" Permission

Site "1" -- "*" Post
User "1" -- "*" Post : author
Post "1" -- "*" Comment
Post "1" -- "*" Media
User "1" -- "*" Media : uploader

ContentLibrary "1" -- "*" LibraryAsset
LibraryAsset "1" -- "*" AssetVersion

Comment ..> SpamService : uses
Post ..> PluginHook : executes hooks
Media ..> StorageService : stored via

@enduml
```
![img_4.png](static-attachments/img_4.png)
---

# Activity Diagram

```plantuml
@startuml
start

:Authenticate User;

if (Email Verified?) then (yes)
  :Select Site;

  if (Has Publish Permission?) then (yes)
    :Create Draft;
    :Add Text Content;

    if (Add Media?) then (yes)
      if (Private Upload?) then (yes)
        :Store as Site Media;
      else (no)
        :Attach Library Asset Version;
      endif
    endif

    :Execute Plugin Hooks;

    if (Validation Passed?) then (yes)
      :Publish Post;
      :Save to Database;
      :Invalidate Cache;
    else (no)
      :Return Error;
    endif

  else (no)
    :Access Denied;
  endif

else (no)
  :Redirect to Email Verification;
endif

stop
@enduml
```
---

![img_1.png](static-attachments/img_1.png)
![img_2.png](static-attachments/img_2.png)

# Sequence Diagram

```plantuml
@startuml

actor Visitor
participant Browser
participant CommentController
participant CommentService
participant SpamService
participant CommentRepository
participant Cache
database DB

Visitor -> Browser : Submit Comment
Browser -> CommentController : POST /comment
CommentController -> CommentService : validate()

CommentService -> SpamService : checkSpam()
SpamService --> CommentService : result

alt Not Spam
    CommentService -> CommentRepository : save(comment)
    CommentRepository -> DB : INSERT
    DB --> CommentRepository : OK

    CommentService -> Cache : invalidate(postComments)

    CommentService --> CommentController : success (Pending)
    CommentController --> Browser : Show Pending Status

else Spam
    CommentService --> CommentController : reject
    CommentController --> Browser : Show Error
end

@enduml
```
![img_3.png](static-attachments/img_3.png)

---

# Use Case Diagram

```plantuml
@startuml
left to right direction

actor Visitor
actor "Registered User" as User
actor Administrator
actor "Email Service" as EmailService
actor "Domain Validation Service" as DomainService

rectangle "CMS Platform" {

  Visitor --> (Register)
  Visitor --> (Login)

  User --> (Create New Site)
  User --> (Verify Email)

  (Register) --> EmailService
  (Verify Email) --> EmailService

  (Create New Site) --> DomainService
  (Create New Site) --> (Assign Owner Role)

  Administrator --> (Manage Content Library)
  Administrator --> (Install Plugin)
}

@enduml
```

![img.png](static-attachments/img.png)