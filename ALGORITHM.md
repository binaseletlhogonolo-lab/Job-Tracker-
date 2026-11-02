# Tracklist: Algorithm and Pseudocode

## 1. Overall flow
1. START app. Load session from localStorage (null if none).
2. Match the URL to a route:
   - Public: `/`, `/login`, `/register`
   - Protected: `/home`, `/jobs/new`, `/jobs/:id`, `/jobs/:id/edit`
   - Anything else: 404 page
3. Protected route and no session: redirect to `/login`, remembering the requested path.
4. Login or register succeeds: save session, go to the remembered path or `/home`.
5. Home reads `q`, `status`, `sort` from the URL, loads the user's jobs, filters, searches, sorts and renders them.
6. Add, edit and delete call JSON Server, then update the screen.

## 2. Route protection
```
FUNCTION Protected(page):
    IF session IS NULL:
        REDIRECT to /login with state.from = current path
    ELSE:
        RENDER page

FUNCTION GuestOnly(page):
    IF session EXISTS: REDIRECT to /home
    ELSE: RENDER page
```

## 3. Register
```
FUNCTION register(username, password, confirm):
    errors = {}
    IF username NOT MATCH /^[A-Za-z0-9_]{3,20}$/: errors.username = "invalid"
    IF length(password) < 6:                       errors.password = "too short"
    IF confirm != password:                        errors.confirm  = "no match"
    IF errors NOT EMPTY: SHOW errors; RETURN

    existing = GET /users?username=username
    IF any existing (case-insensitive): SHOW "username taken"; RETURN

    user = POST /users {username, password}
    SAVE session {id, username}
    NAVIGATE /home
```

## 4. Login
```
FUNCTION login(username, password):
    IF username OR password EMPTY: SHOW errors; RETURN
    matches = GET /users?username=username
    user = matches.find(u => u.username == username)
    IF user IS NULL OR user.password != password:
        SHOW "Incorrect username or password"; RETURN
    SAVE session {id, username}
    NAVIGATE to state.from OR /home
```

## 5. Home list: search, filter, sort via URL
```
FUNCTION Home():
    params = read URL query string
    q      = params.q OR ""
    status = params.status IF status IN [Applied, Interviewed, Rejected] ELSE ""
    sort   = "asc" IF params.sort == "asc" ELSE "desc"      // invalid values fall back

    ON mount: jobs = GET /jobs?userId=session.id            // authorization: own jobs only

    FUNCTION visibleJobs():
        result = jobs
        IF status != "": result = result WHERE job.status == status
        IF q != "":
            term = lowercase(trim(q))
            result = result WHERE term IN lowercase(job.company)
                               OR term IN lowercase(job.role)
        SORT result BY job.dateApplied (ascending if sort == "asc", else descending)
        RETURN result

    ON search input change:  set URL param q      (remove param if empty)
    ON filter change:        set URL param status (remove param if empty)
    ON sort change:          set URL param sort   (remove param if "desc")
    // Updating the URL re-runs visibleJobs(), so URL and screen always agree.

    counts = number of jobs per status
    RENDER counts, controls, visibleJobs() (or an empty-state message)
```

## 6. Add and update job
```
FUNCTION validateJob(form):
    errors = {}
    IF company EMPTY:                       errors.company = "required"
    IF role EMPTY:                          errors.role = "required"
    IF status NOT IN allowed statuses:      errors.status = "invalid"
    IF dateApplied EMPTY:                   errors.dateApplied = "required"
    ELSE IF dateApplied > today:            errors.dateApplied = "no future dates"
    IF email given AND NOT valid email:     errors.contactEmail = "invalid"
    IF phone given AND NOT valid phone:     errors.contactPhone = "invalid"
    IF website given AND NOT http(s) URL:   errors.website = "invalid"
    RETURN errors

FUNCTION saveJob(form, id OR null):
    errors = validateJob(form)
    IF errors NOT EMPTY: SHOW errors; RETURN
    form.userId = session.id
    IF id EXISTS: saved = PUT  /jobs/id  form      // update
    ELSE:         saved = POST /jobs     form      // create
    NAVIGATE /jobs/saved.id
```

## 7. View, edit and delete job (with ownership check)
```
FUNCTION loadJob(id):
    job = GET /jobs/id
    IF request fails OR job.userId != session.id: RENDER 404 page
    ELSE RETURN job

FUNCTION deleteJob(job):
    IF user does NOT confirm: RETURN
    DELETE /jobs/job.id
    remove job from screen (Home) OR NAVIGATE /home (Job page)
```

## 8. Complexity
Search, filter and sort run in the browser over one user's jobs: filter and search are O(n), sort is O(n log n) for n jobs.
