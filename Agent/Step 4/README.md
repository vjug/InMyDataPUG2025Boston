# Step 4: Python Driver – Retrieving Cars

## What Changed in This Step

This step adds READ functionality to the database driver, allowing retrieval of car records by registration number. It also introduces Python dataclasses for structured data representation.

### New/Modified Files

- **`requirements.txt`**: No changes (still uses `python-dotenv` and `requests`)
  
- **`OEDatabaseDriver.py`**: Enhanced with:
  - **`Car` dataclass**: A structured data type to represent car records
  - **`get_car()` method**: Retrieves a car by registration via HTTP GET
  - Improved type hints using `Optional[Car]`
  - Updated example usage to test retrieval

### Code Overview

#### New Car Dataclass

```python
@dataclass
class Car:
    def __init__(self, reg: str = "", make: str = "", model: str = "", year: int = 0):
        self.reg = reg
        self.make = make
        self.model = model
        self.year = year
    reg: str
    make: str
    model: str
    year: int
```

This creates a structured type for car data, similar to a TEMP-TABLE definition in ABL.

#### New get_car() Method

```python
def get_car(self, reg: str) -> Optional[Car]:
    """
    Look up a car by registration.
    Returns a Car if found, otherwise None.
    """
    url = f"{BASE_URL}carService"
    headers = {"Accept": "application/json"}
    
    r = requests.get(url, params={"reg": reg}, headers=headers, timeout=10)
    
    if r.status_code == 200:
        data = r.json()
        return Car(
            reg=str(data.get("reg", "")),
            make=str(data.get("make", "")),
            model=str(data.get("model", "")),
            year=int(data.get("year")) if data.get("year") is not None else 0,
        )
    elif r.status_code in (204, 404):
        return None  # Not found
    # ... error handling
```

#### Updated Example Usage

```python
if __name__ == "__main__":
    driver = OEDatabaseDriver()
    car = driver.get_car("GJ24YBR")
    if car:
        print("Found:", car)
    else:
        print("No car found for that reg.")
```

## What This Implements

**HTTP GET Functionality**: This step implements read operations for car records:

- **REST API GET**: Makes HTTP GET requests with query parameters
- **JSON Deserialization**: Parses JSON responses into Python objects
- **Structured Data**: Uses dataclasses to represent database records with proper types
- **Null Handling**: Returns `None` when records aren't found (rather than raising errors)
- **Query Parameters**: Passes the registration number as a URL parameter

The `get_car()` method handles:
- **200 OK**: Car found, return populated `Car` object
- **204/404**: Car not found, return `None`
- **Other codes**: Network or server errors

## How This Helps Build Towards the Final Solution

This step is essential because:

1. **Lookup Operations**: The AI agent needs to look up existing car records when users provide their registration number - this method enables that capability

2. **Data Transfer Objects**: The `Car` dataclass provides:
   - Type safety (fields have specific types)
   - Default values (empty strings and 0 for year)
   - Easy serialization (can convert to/from dictionaries)
   - Clear structure for passing data between functions

3. **Agent Tool Foundation**: The `get_car()` method becomes the backend for the agent's "lookup car by registration" tool

4. **User Experience Flow**: Enables the primary workflow:
   - User provides registration number
   - Agent looks up the car
   - If found: Agent knows the car details
   - If not found: Agent can ask for details to create a new record (using Step 3's `save_car()`)

5. **Complete CRUD Pattern**: Combined with Step 3, we now have:
   - **Create**: `save_car()` (Step 3)
   - **Read**: `get_car()` (Step 4)
   - Update and Delete will come later if needed

## Testing This Step

1. Ensure you have a car record in the database (use Step 3 to create one if needed)

2. Run the script:
   ```powershell
   py OEDatabaseDriver.py
   ```

3. **Expected Output** (if car exists):
   ```
   Found: Car(reg='GJ24YBR', make='Audi', model='A4', year=2018)
   ```

4. **Test Not Found Case**: Modify the script to search for a non-existent registration:
   ```python
   car = driver.get_car("NOTFOUND")
   ```
   
   Expected output:
   ```
   No car found for that reg.
   ```

## Key Concepts for OpenEdge Developers

- **Dataclass ≈ TEMP-TABLE**: The `Car` dataclass is similar to defining a TEMP-TABLE structure:
  ```abl
  DEFINE TEMP-TABLE ttCar
    FIELD reg AS CHARACTER
    FIELD make AS CHARACTER
    FIELD model AS CHARACTER
    FIELD year AS INTEGER.
  ```

- **REST GET ≈ FIND/CAN-FIND**: The `get_car()` method is similar to:
  ```abl
  FIND FIRST Car WHERE Car.reg = cReg NO-ERROR.
  IF AVAILABLE Car THEN ...
  ```

- **Optional Return Type**: `Optional[Car]` means the method returns either a `Car` object or `None` (like checking `AVAILABLE Car` in ABL)

- **Query Parameters**: The `params={"reg": reg}` becomes `?reg=GJ24YBR` in the URL, similar to passing parameters in ABL procedure calls

## What's Next

Step 5 introduces the LiveKit AI agent, which will use both `save_car()` and `get_car()` as tools to interact with users conversationally.
