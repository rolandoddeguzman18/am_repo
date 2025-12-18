```python
import typing
import core

def fetch_servicenow_resolved_tickets(instance_name: typing.Annotated[str, "ServiceNow instance name (e.g. 'dev12345')"]) -> dict:
    """
    Fetch the 10 most recent resolved tickets from ServiceNow incident table.
    
    Args:
        instance_name: ServiceNow instance name (e.g. 'dev12345')
        
    Returns:
        dict: JSON response with status, message, and data containing resolved tickets
    """
    
    def _get_auth_credentials():
        """Get ServiceNow authentication credentials from secret store."""
        try:
            credentials = core.get_secret('servicenow_api')
            if not credentials or 'username' not in credentials or 'password' not in credentials:
                raise ValueError("Invalid ServiceNow credentials format")
            return credentials['username'], credentials['password']
        except Exception as e:
            raise Exception(f"Failed to retrieve ServiceNow credentials: {str(e)}")
    
    def _build_headers():
        """Build HTTP headers for ServiceNow API request."""
        return {
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }
    
    def _construct_url(instance):
        """Construct ServiceNow REST API URL for resolved incidents."""
        base_url = f"https://{instance}.service-now.com"
        endpoint = "/api/now/table/incident"
        params = "?sysparm_query=state=6&sysparm_limit=10&sysparm_order_by=sys_updated_on"
        return f"{base_url}{endpoint}{params}"
    
    # Parameter validation
    if not instance_name or not isinstance(instance_name, str):
        return {
            "status": "error",
            "message": "Invalid instance_name parameter: must be a non-empty string",
            "data": None
        }
    
    if not instance_name.strip():
        return {
            "status": "error", 
            "message": "Instance name cannot be empty or whitespace only",
            "data": None
        }
    
    try:
        import requests
        
        # Get authentication credentials
        username, password = _get_auth_credentials()
        
        # Prepare request components
        url = _construct_url(instance_name.strip())
        headers = _build_headers()
        auth = (username, password)
        
        # Make API request
        response = requests.get(url, headers=headers, auth=auth, timeout=30)
        
        # Check response status
        if response.status_code == 401:
            return {
                "status": "error",
                "message": "Authentication failed: Invalid ServiceNow credentials",
                "data": None
            }
        elif response.status_code == 403:
            return {
                "status": "error", 
                "message": "Access forbidden: Insufficient permissions for ServiceNow API",
                "data": None
            }
        elif response.status_code == 404:
            return {
                "status": "error",
                "message": f"ServiceNow instance '{instance_name}' not found",
                "data": None
            }
        elif not response.ok:
            return {
                "status": "error",
                "message": f"ServiceNow API request failed with status {response.status_code}: {response.text}",
                "data": None
            }
        
        # Parse response
        try:
            data = response.json()
            tickets = data.get('result', [])
            
            return {
                "status": "success",
                "message": f"Successfully retrieved {len(tickets)} resolved tickets from ServiceNow",
                "data": tickets
            }
            
        except ValueError as e:
            return {
                "status": "error",
                "message": f"Failed to parse ServiceNow API response: {str(e)}",
                "data": None
            }
            
    except ImportError:
        return {
            "status": "error",
            "message": "Required module 'requests' not available",
            "data": None
        }
    except Exception as e:
        return {
            "status": "error",
            "message": f"Unexpected error while fetching ServiceNow tickets: {str(e)}",
            "data": None
        }
```