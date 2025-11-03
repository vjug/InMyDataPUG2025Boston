# Step 3: Python Driver – Saving Cars

## What Changed in This Step

This step transforms the simple environment check script into a functional database driver that can create car records in the OpenEdge database via HTTP POST requests.

### New/Modified Files

- **`requirements.txt`**: Updated to include:
  - `python-dotenv`: (from Step 2)
  - `requests`: **NEW** - Python HTTP library for making REST API calls
  
- **`OEDatabaseDriver.py`**: Major evolution from Step 2:
  - Now defines an `OEDatabaseDriver` class
  - Implements the `save_car()` method to POST car data to the OpenEdge service
  - Includes error handling for HTTP requests
  - Handles response status codes (200, 409, etc.)

### Code Overview

The `OEDatabaseDriver` class now includes:

```python
class OEDatabaseDriver:
    def save_car(self, reg: str, make: str, model: str, year: int) -> bool:
        """
        Calls the car service API to save a car.
        """
        url = f"{BASE_URL}carService"
        
        payload = {
            "reg": reg,
            "make": make,
            "model": model,
            "year": str(year)
        }
        
        response = requests.post(url, json=payload, headers=headers)
        
        # Handle different response codes
        if response.status_code == 200:
            return True
        elif response.status_code == 409:
            # Duplicate registration
            return False
        # ... error handling
```

The script includes example usage:

```python
if __name__ == "__main__":
    driver = OEDatabaseDriver()
    success = driver.save_car("GJ24YBR", "Audi", "A4", 2018)
    print("Car saved successfully:", success)
```

## What This Implements

**HTTP POST Functionality**: This step implements the ability to create new car records:

- **REST API Integration**: Makes HTTP POST requests to the OpenEdge WEB transport service
- **JSON Serialization**: Converts Python dictionaries to JSON payloads
- **Response Handling**: Parses HTTP status codes and response bodies
- **Error Management**: Handles various failure scenarios (network errors, conflicts, unexpected responses)

The `save_car()` method handles:
- **200 OK**: Car successfully created
- **409 Conflict**: Car with this registration already exists
- **Other codes**: Unexpected errors

## How This Helps Build Towards the Final Solution

This step is critical because:

1. **Database Write Operations**: Establishes the pattern for modifying database records that will be used by the AI agent when users want to register new vehicles

2. **REST API Pattern**: Demonstrates the request/response cycle that will be repeated for all database operations:
   - Build a payload
   - Send HTTP request
   - Handle response
   - Return result

3. **Error Handling Foundation**: Introduces proper error handling that ensures the AI agent can gracefully handle:
   - Network failures
   - Duplicate records
   - Service unavailability

4. **Agent Tool Foundation**: The `save_car()` method will become a tool that the AI agent can call when a user needs to register a new vehicle

5. **Testability**: The `if __name__ == "__main__"` pattern allows testing the driver independently before integrating with the agent

## Testing This Step

1. Ensure your OpenEdge service is running with the `carHandler` and `carService` deployed

2. Run the script:
   ```powershell
   py OEDatabaseDriver.py
   ```

3. **Expected Output**: 
   ```
   Car saved successfully: True
   ```

4. **Verify in Database**: Use the `OpenEdge\ViewCars.w` tool to confirm the car record was created with:
   - Registration: GJ24YBR
   - Make: Audi
   - Model: A4
   - Year: 2018

5. **Test Duplicate Handling**: Run the script again - it should return `False` and print a conflict message

## Key Concepts for OpenEdge Developers

- **REST POST ≈ CREATE Operation**: Similar to creating a new record with `CREATE Car` in ABL
- **JSON Payload**: Similar to a temp-table with defined fields
- **HTTP Status Codes**: 
  - 200 = Success (like return value TRUE)
  - 409 = Conflict (like CAN-FIND checking for duplicates)
  - 500 = Server error (like catching ERROR conditions)
- **Class Method**: The `save_car()` method is like a FUNCTION in ABL that returns a LOGICAL value

## What's Next

Step 4 will add the complementary READ operation (`get_car()`) to retrieve existing car records from the database.
