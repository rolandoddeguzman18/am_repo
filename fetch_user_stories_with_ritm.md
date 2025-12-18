# Fetch User Stories with RITM Function

```python
import requests
from typing import Dict, Any, List, Optional
from typing_extensions import Annotated
import core


def fetch_user_stories_with_ritm(
    ritm_key: Annotated[str, "RITM key to search for in Jira stories"] = "RITM321312",
    max_results: Annotated[int, "Maximum number of results to return"] = 50
) -> Dict[str, Any]:
    """
    Fetch Jira user stories containing the specified RITM key.
    
    Searches for user stories that contain the RITM key in their summary, 
    description, or custom fields using Jira REST API.
    
    Args:
        ritm_key: The RITM key to search for (default: "RITM321312")
        max_results: Maximum number of results to return (default: 50)
        
    Returns:
        Dict containing:
        - status: "success" or "error"
        - message: Description of the operation result
        - data: List of user stories or None if error
        
    Example:
        >>> result = fetch_user_stories_with_ritm("RITM321312")
        >>> if result["status"] == "success":
        ...     for story in result["data"]:
        ...         print(f"Story: {story['key']} - {story['summary']}")
    """
    
    def _validate_parameters() -> Optional[str]:
        """Validate input parameters."""
        if not ritm_key or not isinstance(ritm_key, str):
            return "RITM key must be a non-empty string"
        if not isinstance(max_results, int) or max_results <= 0:
            return "max_results must be a positive integer"
        return None
    
    def _get_jira_credentials() -> Dict[str, str]:
        """Retrieve Jira API credentials from core secret."""
        try:
            secret = core.get_secret('jira_api')
            required_fields = ['domain', 'email', 'api_token']
            
            for field in required_fields:
                if field not in secret:
                    raise ValueError(f"Missing required field '{field}' in jira_api secret")
                    
            return {
                'domain': secret['domain'],
                'email': secret['email'], 
                'api_token': secret['api_token']
            }
        except Exception as e:
            raise ValueError(f"Failed to retrieve Jira credentials: {str(e)}")
    
    def _build_jql_query() -> str:
        """Build JQL query to search for user stories with RITM key."""
        return (
            f'type = Story AND '
            f'(summary ~ "{ritm_key}" OR '
            f'description ~ "{ritm_key}" OR '
            f'cf[*] ~ "{ritm_key}")'
        )
    
    def _make_jira_request(
        url: str, 
        auth: tuple, 
        params: Dict[str, Any]
    ) -> requests.Response:
        """Make authenticated request to Jira API."""
        headers = {
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }
        
        response = requests.get(
            url, 
            headers=headers, 
            auth=auth, 
            params=params,
            timeout=30
        )
        response.raise_for_status()
        return response
    
    def _format_story_data(story: Dict[str, Any]) -> Dict[str, Any]:
        """Format individual story data for response."""
        fields = story.get('fields', {})
        return {
            'key': story.get('key'),
            'id': story.get('id'),
            'summary': fields.get('summary'),
            'description': fields.get('description'),
            'status': fields.get('status', {}).get('name'),
            'assignee': fields.get('assignee', {}).get('displayName') if fields.get('assignee') else None,
            'priority': fields.get('priority', {}).get('name'),
            'created': fields.get('created'),
            'updated': fields.get('updated'),
            'story_points': fields.get('customfield_10016'),  # Common story points field
            'url': f"https://{credentials['domain']}/browse/{story.get('key')}"
        }
    
    # Validate parameters
    validation_error = _validate_parameters()
    if validation_error:
        return {
            "status": "error",
            "message": f"Parameter validation failed: {validation_error}",
            "data": None
        }
    
    try:
        # Get credentials
        credentials = _get_jira_credentials()
        
        # Build request components
        base_url = f"https://{credentials['domain']}/rest/api/2/search"
        auth = (credentials['email'], credentials['api_token'])
        jql_query = _build_jql_query()
        
        # Request parameters
        params = {
            'jql': jql_query,
            'maxResults': max_results,
            'fields': 'summary,description,status,assignee,priority,created,updated,customfield_10016',
            'expand': 'changelog'
        }
        
        # Make API request
        response = _make_jira_request(base_url, auth, params)
        response_data = response.json()
        
        # Process results
        stories = []
        for issue in response_data.get('issues', []):
            formatted_story = _format_story_data(issue)
            stories.append(formatted_story)
        
        total_found = response_data.get('total', 0)
        returned_count = len(stories)
        
        return {
            "status": "success",
            "message": f"Found {total_found} user stories containing '{ritm_key}'. Returned {returned_count} results.",
            "data": {
                "stories": stories,
                "total_found": total_found,
                "returned_count": returned_count,
                "ritm_key": ritm_key
            }
        }
        
    except requests.exceptions.RequestException as e:
        return {
            "status": "error", 
            "message": f"Jira API request failed: {str(e)}",
            "data": None
        }
    except ValueError as e:
        return {
            "status": "error",
            "message": str(e),
            "data": None
        }
    except Exception as e:
        return {
            "status": "error",
            "message": f"Unexpected error occurred: {str(e)}",
            "data": None
        }
```