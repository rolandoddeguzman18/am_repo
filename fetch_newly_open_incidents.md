# ServiceNow Incident Fetcher

```python
import json
import core

def fetch_newly_open_incidents(servicenow_instance_url):
    """
    Fetches the 10 most recently opened, active incidents from ServiceNow via REST API.
    
    Args:
        servicenow_instance_url (str): The base URL of the ServiceNow instance
                                     (e.g., 'https://your-instance.servicenow.com')
    
    Returns:
        str: JSON string with status, message, and data keys for success
        dict: Dictionary with status and message keys for errors
    
    Example:
        >>> result = fetch_newly_open_incidents('https://dev123456.servicenow.com')
        >>> print(result)
        '{"status": "success", "message": "Successfully fetched 10 incidents", "data": [...]}'
    """
    import requests
    
    # Parameter validation
    if not servicenow_instance_url or not isinstance(servicenow_instance_url, str):
        return {
            'status': 'error',
            'message': 'Invalid ServiceNow instance URL provided'
        }
    
    # Remove trailing slash if present
    base_url = servicenow_instance_url.rstrip('/')
    
    def get_auth_credentials():
        """Helper function to get ServiceNow credentials securely."""
        try:
            credentials = core.get_secret('servicenow')
            if not credentials or 'username' not in credentials or 'password' not in credentials:
                raise ValueError("Missing required credentials")
            return credentials['username'], credentials['password']
        except Exception as e:
            raise Exception(f"Failed to retrieve credentials: {str(e)}")
    
    def get_request_headers():
        """Helper function to get standard request headers."""
        return {
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }
    
    try:
        # Get authentication credentials
        username, password = get_auth_credentials()
        
        # Build API endpoint URL
        endpoint = f"{base_url}/api/now/table/incident"
        
        # Query parameters for 10 most recent active incidents
        params = {
            'sysparm_query': 'active=true^state=1',  # Active and New state
            'sysparm_order_by': '^ORDERBYDESCopened_at',  # Most recent first
            'sysparm_limit': '10',
            'sysparm_fields': 'number,short_description,state,priority,opened_at,assigned_to,caller_id'
        }
        
        # Make the API request
        response = requests.get(
            endpoint,
            auth=(username, password),
            headers=get_request_headers(),
            params=params,
            timeout=30
        )
        
        # Check if request was successful
        response.raise_for_status()
        
        # Parse JSON response
        data = response.json()
        
        # Extract incidents from response
        incidents = data.get('result', [])
        
        # Return success response as JSON string
        success_response = {
            'status': 'success',
            'message': f'Successfully fetched {len(incidents)} incidents',
            'data': incidents
        }
        
        return json.dumps(success_response)
        
    except requests.exceptions.RequestException as e:
        return {
            'status': 'error',
            'message': f'HTTP request failed: {str(e)}'
        }
    except ValueError as e:
        return {
            'status': 'error',
            'message': f'JSON parsing error: {str(e)}'
        }
    except Exception as e:
        return {
            'status': 'error',
            'message': f'Unexpected error occurred: {str(e)}'
        }
```