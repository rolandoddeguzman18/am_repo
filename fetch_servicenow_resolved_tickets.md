# fetch_servicenow_resolved_tickets

```python
import requests
import typing
import json
import core


def fetch_servicenow_resolved_tickets(
    instance_url: typing.Annotated[str, "ServiceNow instance URL"], 
    secret_name: typing.Annotated[str, "Core secret name for credentials"]
) -> str:
    """
    Fetch the last 10 resolved ServiceNow incidents.
    
    This function retrieves resolved incidents (state=6) from ServiceNow using the Table API.
    It fetches credentials from the core secret management system and returns structured
    ticket data including number, short description, state, and resolution details.
    
    Args:
        instance_url: The ServiceNow instance URL (e.g., 'https://company.service-now.com')
        secret_name: Name of the secret in core containing ServiceNow credentials
        
    Returns:
        JSON string containing list of resolved tickets on success, or error dict on failure
        
    Raises:
        ValueError: If parameters are invalid
        requests.RequestException: If API call fails
        Exception: For other unexpected errors
    """
    
    def _validate_parameters(instance_url: str, secret_name: str) -> None:
        """Validate input parameters."""
        if not instance_url or not isinstance(instance_url, str):
            raise ValueError("instance_url must be a non-empty string")
        if not secret_name or not isinstance(secret_name, str):
            raise ValueError("secret_name must be a non-empty string")
        if not instance_url.startswith(('http://', 'https://')):
            raise ValueError("instance_url must start with http:// or https://")
    
    def _get_credentials(secret_name: str) -> typing.Dict[str, str]:
        """Retrieve ServiceNow credentials from core."""
        try:
            secret_data = core.get_secret(secret_name)
            if not secret_data or 'username' not in secret_data or 'password' not in secret_data:
                raise ValueError("Secret must contain 'username' and 'password' fields")
            return secret_data
        except Exception as e:
            raise ValueError(f"Failed to retrieve credentials: {str(e)}")
    
    def _build_request_config(credentials: typing.Dict[str, str]) -> typing.Dict[str, typing.Any]:
        """Build authentication and headers for API request."""
        return {
            'auth': (credentials['username'], credentials['password']),
            'headers': {
                'Accept': 'application/json',
                'Content-Type': 'application/json'
            },
            'timeout': 30
        }
    
    def _make_api_request(instance_url: str, config: typing.Dict[str, typing.Any]) -> typing.Dict[str, typing.Any]:
        """Make the ServiceNow API request for resolved incidents."""
        endpoint = f"{instance_url.rstrip('/')}/api/now/table/incident"
        params = {
            'sysparm_query': 'state=6',  # Resolved state
            'sysparm_limit': 10,
            'sysparm_fields': 'number,short_description,state,sys_created_on,resolved_at,resolved_by,resolution_notes,priority,category,subcategory',
            'sysparm_order_by': '-resolved_at'
        }
        
        response = requests.get(endpoint, params=params, **config)
        response.raise_for_status()
        return response.json()
    
    def _format_ticket_data(api_response: typing.Dict[str, typing.Any]) -> typing.List[typing.Dict[str, typing.Any]]:
        """Format the API response into structured ticket data."""
        tickets = []
        for record in api_response.get('result', []):
            ticket = {
                'number': record.get('number', ''),
                'short_description': record.get('short_description', ''),
                'state': record.get('state', ''),
                'created_on': record.get('sys_created_on', ''),
                'resolved_at': record.get('resolved_at', ''),
                'resolved_by': record.get('resolved_by', ''),
                'resolution_notes': record.get('resolution_notes', ''),
                'priority': record.get('priority', ''),
                'category': record.get('category', ''),
                'subcategory': record.get('subcategory', '')
            }
            tickets.append(ticket)
        return tickets
    
    try:
        # Validate input parameters
        _validate_parameters(instance_url, secret_name)
        
        # Get credentials from core
        credentials = _get_credentials(secret_name)
        
        # Build request configuration
        request_config = _build_request_config(credentials)
        
        # Make API request
        api_response = _make_api_request(instance_url, request_config)
        
        # Format and return results
        formatted_tickets = _format_ticket_data(api_response)
        
        result = {
            'success': True,
            'tickets': formatted_tickets,
            'total_count': len(formatted_tickets)
        }
        
        return json.dumps(result, indent=2)
        
    except ValueError as e:
        error_result = {
            'success': False,
            'error': 'validation_error',
            'message': str(e)
        }
        return error_result
        
    except requests.RequestException as e:
        error_result = {
            'success': False,
            'error': 'api_error',
            'message': f"ServiceNow API request failed: {str(e)}"
        }
        return error_result
        
    except Exception as e:
        error_result = {
            'success': False,
            'error': 'unexpected_error',
            'message': f"Unexpected error occurred: {str(e)}"
        }
        return error_result
```