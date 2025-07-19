# CONNECT.IN
"the project i bulid using python in which user able see the profile around its 10 km radius ,he can send follow request ,accept,text and intesrting feature is having the "THE CONNFESTION BOX "any one can write blogs ,post pic ,post videoes .The project nothing but "REGIONAL SPECIFIC SOCIAL MEDIA" 


the code is the follows :-----

from fastapi import FastAPI, HTTPException, Body, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from database import user_collection, requests_collection
from typing import Optional
from bson import ObjectId
from passlib.context import CryptContext
from jose import jwt, JWTError
from datetime import datetime, timedelta
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

# CORS for frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models
class UserModel(BaseModel):
    username: str
    password: str
    lat: float
    lng: float

class LoginModel(BaseModel):
    username: str
    password: str

class LocationQuery(BaseModel):
    lat: float
    lng: float

class FriendRequestModel(BaseModel):
    to_username: str

# Password Hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# JWT Settings
SECRET_KEY = "connectin_secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

def create_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload.get("sub")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/signup")
async def signup(user: UserModel = Body(...)):
    existing = await user_collection.find_one({"username": user.username})
    if existing:
        raise HTTPException(status_code=400, detail="Username already exists")

    hashed_password = pwd_context.hash(user.password)
    await user_collection.insert_one({
        "username": user.username,
        "password": hashed_password,
        "lat": user.lat,
        "lng": user.lng
    })
    return {"message": "User created"}

@app.post("/login")
async def login(credentials: LoginModel = Body(...)):
    user = await user_collection.find_one({"username": credentials.username})
    if not user or not pwd_context.verify(credentials.password, user["password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    token = create_token({"sub": credentials.username})
    return {"access_token": token, "token_type": "bearer"}

@app.get("/nearby-users")
async def get_nearby_users(current_user: str = Depends(get_current_user)):
    me = await user_collection.find_one({"username": current_user})
    if not me:
        raise HTTPException(status_code=404, detail="User not found")

    lat1, lon1 = me["lat"], me["lng"]
    nearby = []

    async for user in user_collection.find({"username": {"$ne": current_user}}):
        lat2, lon2 = user["lat"], user["lng"]
        dist = haversine(lat1, lon1, lat2, lon2)
        if dist <= 10:
            nearby.append({
                "username": user["username"],
                "distance": round(dist, 2)
            })
    return {"nearby_users": nearby}

def haversine(lat1, lon1, lat2, lon2):
    from math import radians, sin, cos, sqrt, atan2
    R = 6371
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    lat1 = radians(lat1)
    lat2 = radians(lat2)
    a = sin(dlat / 2) ** 2 + cos(lat1) * cos(lat2) * sin(dlon / 2) ** 2
    c = 2 * atan2(sqrt(a), sqrt(1 - a))
    return R * c
