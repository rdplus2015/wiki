# DRF — APIView & Serializers in Practice

## 1. APIView

`APIView` is the equivalent of a classic Django view, but for a REST API.

It handles:
- automatic JSON parsing
- HTTP status codes
- auth / permissions
- integration with serializers

### 1.1 Test endpoint (health-check)

Used to check that the API is responding.

`GET → /api/health/`

```python
HealthCheck(APIView):
    """
    Simple health-check endpoint.
    """

    def get(self, request, format=None):
        data = {
            "firstRule": "Hello world",
        }
        return Response(data, status=status.HTTP_200_OK)
```

### 1.2 Receiving and validating data

```python
class HelloAPIView(APIView):
    def post(self, request, format=None):
        serializer = HelloSerializer(data=request.data) // creating the serializer // request.data = JSON sent by the client. DRF converts it to a Python dict.
        serializer.is_valid(raise_exception=True) // validation: if invalid → DRF automatically returns a 400 error: // No need to handle try/catch.
        name = serializer.validated_data["name"] // getting clean data // validated_data = cleaned + secured data.
        age = serializer.validated_data["age"]
        // JSON response
        return Response(
            {
            "hello": name,
            "age": age
            },
            status=status.HTTP_200_OK
        )
```

**Walkthrough:**

1. `HelloSerializer(data=request.data)` — creates the serializer. `request.data` is the JSON sent by the client; DRF converts it to a Python dict.
2. `serializer.is_valid(raise_exception=True)` — validation. If the data is invalid, DRF automatically returns a 400 error; no need for try/catch.
3. `serializer.validated_data["name"]` / `["age"]` — retrieving the validated, cleaned data.
4. `Response(...)` — builds the JSON response.

---

## 2. Serializers

A serializer is used to:
- validate incoming data (JSON → Python)
- clean the data
- expose a usable format via `validated_data`
- convert to JSON on output when needed

### 2.1 Example with custom validation

```python
from rest_framework import serializers


class HelloSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=50)
    age = serializers.IntegerField()

    def validate_name(self, value):
        value =  value.strip()
        if len(value) < 2:
            raise serializers.ValidationError("Name must be between 2 and 20 characters")
        return value
```

**`class HelloSerializer(serializers.Serializer)`**
A simple serializer, not tied to a database model. Used to validate JSON received in a request.

**`name = serializers.CharField(max_length=50)`**
Expected field in the JSON:
```json
{
  "name": "Ridi"
}
```
Automatic validation: must be a string, max 50 characters, required by default.

**`def validate_name(self, value)`**
Custom validation on the `name` field. DRF automatically calls this method if it exists on the serializer.

Logic:
- `strip()` removes leading/trailing whitespace
- if the length is less than 2 characters → error
- otherwise, returns the cleaned value

On error, DRF automatically returns:
```json
{
  "name": ["Name must be at least 2 characters."]
}
```
