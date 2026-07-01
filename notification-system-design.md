# Notification System Design

## Stage 1: REST API Design

The notification platform needs to let a student see their notifications when they log in. The main actions are:

- Get list of notifications
- Get one notification
- Create a notification
- Mark a notification as read
- Delete a notification
- Get unread count

**Naming rules used:**
- URLs use plural nouns, like `/notifications`
- Use proper HTTP methods: GET, POST, PATCH, DELETE
- JSON fields use camelCase
- Use proper status codes: 200, 201, 204, 404

**Headers used:**
- `Authorization: Bearer <token>` – to know which student is logged in
- `Content-Type: application/json`
- `Accept: application/json`

**Notification JSON structure:**
```json
{
  "id": "1",
  "studentId": "1042",
  "notificationType": "PLACEMENT",
  "title": "Infosys Drive",
  "message": "Placement drive on July 5",
  "isRead": false,
  "createdAt": "2026-06-28T09:00:00Z"
}
```

**Endpoints:**

1. `GET /api/v1/notifications` – get list of notifications (supports filters like isRead, page, limit)
2. `GET /api/v1/notifications/{id}` – get one notification
3. `POST /api/v1/notifications` – create a notification
4. `PATCH /api/v1/notifications/{id}/read` – mark one notification as read
5. `DELETE /api/v1/notifications/{id}` – delete a notification
6. `GET /api/v1/notifications/unread-count` – get unread count

Example response for creating a notification:
```json
{
  "id": "5",
  "studentId": "1042",
  "notificationType": "RESULT",
  "title": "Result Published",
  "message": "Your semester result is out",
  "isRead": false
}
```

---

## Stage 2: Database Design

**Which database?**
I would use a relational database (MySQL/PostgreSQL) instead of NoSQL.

Reasons:
- Notification data has a fixed structure (same fields every time)
- Notifications belong to students, so it needs relationships
- Marking as read needs safe and consistent updates
- We need to filter and sort data often, which SQL handles well

**Schema:**
```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE notifications (
    id INT PRIMARY KEY AUTO_INCREMENT,
    student_id INT,
    notification_type VARCHAR(20),
    title VARCHAR(200),
    message TEXT,
    is_read BOOLEAN DEFAULT false,
    created_at DATETIME,
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);
```

**Problems as data grows:**
- Queries become slow without proper indexes
- Too many indexes slow down inserts and updates
- Old data keeps piling up and makes the table bigger and slower
- Table becomes hard to maintain

**How to solve it:**
- Add indexes only on columns actually used in WHERE/ORDER BY
- Archive old notifications (like older than 6 months) to another table
- Split the table by date (partitioning)
- Use caching so we don't hit the database every time

**Sample queries:**
```sql
-- Get unread notifications of a student
SELECT id, notification_type, title, is_read, created_at
FROM notifications
WHERE student_id = 1042 AND is_read = false
ORDER BY created_at DESC;

-- Mark as read
UPDATE notifications
SET is_read = true
WHERE id = 5;

-- Unread count
SELECT COUNT(*) FROM notifications
WHERE student_id = 1042 AND is_read = false;
```

---

## Stage 3: Query Optimization

**Given query:**
```sql
SELECT * FROM notifications
WHERE studentID = 1042 AND isRead = false
ORDER BY createdAt DESC;
```

**Is it correct?**
Yes, it gives the right result. The problem is speed, not correctness.

**Why is it slow?**
- `SELECT *` fetches all columns even if not needed
- No index on studentID and isRead, so database scans all 5,000,000 rows
- Sorting by createdAt also takes extra time since there's no index for it

**Computation cost:**
Without index, it checks all rows one by one, so cost is about O(n), where n = 5,000,000. This gets worse as more notifications are added.

**Is "index every column" good advice?**
No, this is not a good idea.
- Every extra index slows down INSERT and UPDATE operations
- Indexes take extra storage
- Indexing something like isRead (only true/false) barely helps

**Better solution:**
Create one combined (composite) index that matches how the query is actually used:
```sql
CREATE INDEX idx_student_unread_created
ON notifications (studentID, isRead, createdAt DESC);
```

Also avoid SELECT *:
```sql
SELECT id, notificationType, title, isRead, createdAt
FROM notifications
WHERE studentID = 1042 AND isRead = false
ORDER BY createdAt DESC;
```

**Query: unread placement notifications in last 7 days**
```sql
SELECT id, studentID, notificationType, title, isRead, createdAt
FROM notifications
WHERE notificationType = 'Placement'
  AND isRead = false
  AND createdAt >= NOW() - INTERVAL 7 DAY
ORDER BY createdAt DESC;
```

Supporting index:
```sql
CREATE INDEX idx_type_unread_created
ON notifications (notificationType, isRead, createdAt DESC);
```

---

## Stage 4: Reducing Database Load

**Problem:**
Every time a student opens a page, the app fetches notifications from the database again. With many students and many page loads, this puts too much load on the database and makes the app slow.

**Solutions:**

1. **Caching (Redis):** Store recent notifications and unread count in cache for a short time (like 30-60 seconds). Page loads read from cache instead of database.
   - Tradeoff: Data may be a few seconds old, but this is acceptable for notifications.

2. **Store unread count separately:** Keep an `unread_count` column on the student table instead of counting every time.
   - Tradeoff: Needs to be updated carefully whenever a notification is read or created, or it can go out of sync.

3. **Pagination:** Only load a small number of notifications at a time (like 20), not the full history.
   - This should always be used, no real downside.

4. **Real-time updates (WebSockets):** Instead of the student refreshing the page again and again, push new notifications directly to them.
   - Tradeoff: More complex to build and maintain, but greatly reduces repeated database calls.

**Best approach:**
Start with pagination + caching + storing unread count separately, since these are simple and give good results. If needed later, add real-time updates using WebSockets for a better user experience.
