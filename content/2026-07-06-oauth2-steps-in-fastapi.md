Title: Locking the Gate: A Practical Walkthrough of OAuth2 in FastAPI
Date: 2026-07-06
Category: Article
Tags: oauth2, fastapi, api
Slug: oauth2-authentication-step-by-step

Security is one of those things every API needs and almost nobody enjoys building. FastAPI makes it far less painful than most frameworks, thanks to built-in support for OAuth2 flows, dependency injection, and automatic docs that even show you a working "Authorize" button. This article walks through building a complete OAuth2 password-flow authentication system in FastAPI — from hashing passwords to issuing JWTs to protecting endpoints — with working code at every stage.

We'll build the **OAuth2 Password Bearer flow**, the most common pattern for first-party APIs (as opposed to third-party login like "Sign in with Google," which uses the Authorization Code flow instead).


# **Step 1: What OAuth2 Actually Buys You Here**

OAuth2 is a framework for delegated authorization, not a single algorithm. FastAPI's OAuth2PasswordBearer implements one specific piece of it: the client sends a username and password to a token endpoint, receives an access token (typically a JWT), and then presents that token as a Bearer header on every subsequent request. FastAPI never sees the password again after login — only the token.

This gets you:

1. A standard Authorization: Bearer <token> header, understood by every HTTP client and API tool

2. Stateless verification (no session store needed if you use JWTs)

3. Automatic integration with Swagger UI's login form


# **Step 2: Project Setup**

Install the dependencies you'll need:

bash

    pip install fastapi uvicorn "python-jose[cryptography]" "passlib[bcrypt]" python-multipart

python-jose — encodes/decodes JWTs
passlib[bcrypt] — securely hashes passwords
python-multipart — required for FastAPI to parse form data (the login endpoint expects application/x-www-form-urlencoded)

Project structure for this walkthrough:

    app/
    ├── main.py
    ├── auth.py
    ├── models.py
    └── database.py


# **Step 3: Modeling Users and "Storage"**

To keep the focus on authentication, we'll use an in-memory dictionary instead of a real database. Swapping in SQLAlchemy or an async ORM later doesn't change anything below.

python

    # models.py
    from pydantic import BaseModel

    class User(BaseModel):
        username: str
        email: str | None = None
        full_name: str | None = None
        disabled: bool | None = None

    class UserInDB(User):
        hashed_password: str

    class Token(BaseModel):
        access_token: str
        token_type: str

    class TokenData(BaseModel):
        username: str | None = None

python

    # database.py
    from .auth import get_password_hash

    fake_users_db = {
        "jdoe": {
            "username": "jdoe",
            "full_name": "Jane Doe",
            "email": "jdoe@example.com",
            "hashed_password": get_password_hash("secret123"),
            "disabled": False,
        }
    }


# **Step 4: Hashing Passwords**

Never store plaintext passwords. passlib with bcrypt handles hashing and verification:

python

    # auth.py (part 1)
    from passlib.context import CryptContext

    pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

    def verify_password(plain_password: str, hashed_password: str) -> bool:
        return pwd_context.verify(plain_password, hashed_password)

    def get_password_hash(password: str) -> str:
        return pwd_context.hash(password)


# **Step 5: Issuing JWT Access Tokens**

Now add JWT creation logic. The token payload (the "claims") should be minimal — just enough to identify the user and set an expiry.

python

    # auth.py (part 2)
    from datetime import datetime, timedelta, timezone
    from jose import JWTError, jwt

    SECRET_KEY = "CHANGE_ME_TO_A_LONG_RANDOM_VALUE"  # use env var in production
    ALGORITHM = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES = 30

    def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
        to_encode = data.copy()
        expire = datetime.now(timezone.utc) + (
            expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
        )
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

Never hardcode SECRET_KEY in real code. Generate one with openssl rand -hex 32 and load it from an environment variable.


# **Step 6: Authenticating the User**

A helper to look up a user and verify their password:

python

    # auth.py (part 3)
    from .models import UserInDB
    from .database import fake_users_db

    def get_user(db, username: str) -> UserInDB | None:
        if username in db:
            return UserInDB(**db[username])
        return None

    def authenticate_user(db, username: str, password: str):
        user = get_user(db, username)
        if not user or not verify_password(password, user.hashed_password):
            return False
        return user


# **Step 7: The Login Endpoint (Token Route)**

This is where OAuth2PasswordRequestForm comes in — it expects username and password as form fields, matching the OAuth2 spec.

python
    
    # main.py (part 1)
    from datetime import timedelta
    from fastapi import FastAPI, Depends, HTTPException, status
    from fastapi.security import OAuth2PasswordRequestForm

    from .auth import authenticate_user, create_access_token, ACCESS_TOKEN_EXPIRE_MINUTES
    from .database import fake_users_db
    from .models import Token

    app = FastAPI()

    @app.post("/token", response_model=Token)
    async def login_for_access_token(form_data: OAuth2PasswordRequestForm = Depends()):
        user = authenticate_user(fake_users_db, form_data.username, form_data.password)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Incorrect username or password",
                headers={"WWW-Authenticate": "Bearer"},
            )
        access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
        access_token = create_access_token(
            data={"sub": user.username}, expires_delta=access_token_expires
        )
        return {"access_token": access_token, "token_type": "bearer"}

The "sub" claim (subject) is the JWT-standard field for identifying who the token belongs to.


# **Step 8: Protecting Routes with a Dependency**

This is where FastAPI's dependency injection shines. OAuth2PasswordBearer tells FastAPI where clients get tokens from (tokenUrl="token") and doubles as a dependency that extracts the token from the Authorization header automatically.

python

    # main.py (part 2)
    from fastapi.security import OAuth2PasswordBearer
    from jose import JWTError, jwt

    from .auth import SECRET_KEY, ALGORITHM
    from .database import fake_users_db
    from .models import User, TokenData
    from .auth import get_user

    oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

    async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
        credentials_exception = HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )   
        try:    
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            username: str = payload.get("sub")
            if username is None:
                raise credentials_exception
            token_data = TokenData(username=username)
        except JWTError:
            raise credentials_exception

        user = get_user(fake_users_db, username=token_data.username)
        if user is None:
            raise credentials_exception
        return user

    async def get_current_active_user(current_user: User = Depends(get_current_user)) -> User:
        if current_user.disabled:
            raise HTTPException(status_code=400, detail="Inactive user")
        return current_user

Now any route can require authentication just by depending on get_current_active_user:

python

    # main.py (part 3)
    @app.get("/users/me", response_model=User)
    async def read_users_me(current_user: User = Depends(get_current_active_user)):
        return current_user


# **Step 9: Trying It Out**

Run the server:

bash

    uvicorn app.main:app --reload

Visit http://127.0.0.1:8000/docs. FastAPI's Swagger UI shows an **Authorize** button because it detects the OAuth2PasswordBearer scheme. Click it, enter jdoe / secret123, and Swagger automatically attaches the resulting bearer token to every subsequent request you try in the docs.

From the command line, the same flow looks like this:

bash

    curl -X POST http://127.0.0.1:8000/token \
        -d "username=jdoe&password=secret123"

    # → {"access_token": "eyJhbGciOi...", "token_type": "bearer"}

    curl http://127.0.0.1:8000/users/me \
        -H "Authorization: Bearer eyJhbGciOi..."

    # → {"username": "jdoe", "email": "jdoe@example.com", ...}


# **Step 10: Production Considerations**

A few things worth addressing before shipping this:

1. **Secret management** — pull SECRET_KEY from environment variables or a secrets manager, never source code.

2. **Token expiry & refresh tokens** — short-lived access tokens (15–30 min) paired with longer-lived refresh tokens reduce the damage from a leaked token.

3. **HTTPS everywhere** — bearer tokens are equivalent to passwords in transit; never send them over plain HTTP.

4. **Password policy** — bcrypt handles storage safely, but pair it with reasonable password strength rules at signup.

5. **Real persistence** — swap the in-memory dict for a database with proper indexing on username/email and unique constraints.

6. **Scopes** — if you need fine-grained permissions, OAuth2PasswordBearer supports scopes so tokens can carry roles like "items:read" or "admin".


# **Wrapping Up**

FastAPI doesn't hide OAuth2's moving parts behind magic — it gives you clear, composable pieces: a scheme that extracts tokens, dependencies that verify them, and Pydantic models that keep everything typed. Once the pattern clicks, adding authentication to a new route is a one-line Depends() away.