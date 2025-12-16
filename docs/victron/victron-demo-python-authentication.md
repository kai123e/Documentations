# Victron Demo Account Authentication

This note walks through calling the Victron Energy VRM demo endpoint, decoding the returned JWT, and printing the demo user ID (UID) with concise Python helper functions.

## Requirements

- Python 3
- `requests` library for HTTPS calls (install with pip)
- Internet access to `https://vrmapi.victronenergy.com`

=== "Terminal"
```bash
pip install requests
```

## Shared Imports and Constants

=== "Python"
```python
import base64
import json
from datetime import datetime
import requests

BASE_URL = "https://vrmapi.victronenergy.com/v2"
```

## Decode the JWT Payload

=== "Python"
```python
def decodeJwt(token: str) -> dict[str, int]:
    """Decode the JWT payload and return the UID."""

    try:
        header_segment, payload_segment, signature_segment = token.split('.')
    except ValueError as exc:
        raise ValueError('Invalid JWT token') from exc

    padded_payload = payload_segment + '=' * (-len(payload_segment) % 4)

    try:
        payload_bytes = base64.urlsafe_b64decode(padded_payload.encode('ascii'))
        payload = json.loads(payload_bytes.decode('utf-8'))
    except (ValueError, json.JSONDecodeError) as exc:
        raise ValueError('Failed to decode JWT token') from exc

    if 'uid' not in payload:
        raise ValueError('JWT payload does not contain uid')

    return {'uid': payload['uid']}
```

## Call the Demo Login Endpoint

=== "Python"
```python
def loginAsDemo() -> dict[str, str]:
    try:
        response = requests.get(f'{BASE_URL}/auth/loginAsDemo')
        if response.status_code == 200:
            data = response.json()
            return data
        else:
            print("Error:", response.status_code)

    except requests.exceptions.RequestException as e:
        print("Request failed:", e)
```

## Putting It All Together

=== "Python"
```python
def main() -> None:
    token_data = loginAsDemo()

    if not token_data or 'token' not in token_data:
        print('Failed to obtain demo token')
        return

    token = token_data['token']

    try:
        user = decodeJwt(token)
    except ValueError as exc:
        print(f'JWT decode error: {exc}')
        return

    print('Demo login succeeded:')
    print(f"  Token issued at: {datetime.utcnow():%Y-%m-%d %H:%M:%S} UTC")
    print(f"  UID: {user['uid']}")
    print(f'  Raw JWT token: {token}')


if __name__ == "__main__":
    main()
```

Run `python3 victron_demo.py` and you should see the decoded payload along with the UID. Demo tokens expire quickly, so request a fresh token whenever you need to re-authenticate.
