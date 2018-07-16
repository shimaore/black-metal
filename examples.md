```
Message  | Transfer-disposition | Endpoint-disposition | Matches

On-hook agent answers then blind-transfers an incoming call to another line

UNBRIDGE | (none)               | ANSWER               | is_remote_call
HANGUP   | recv_replace         | BLIND_TRANSFER       | is_onhook_call



```
