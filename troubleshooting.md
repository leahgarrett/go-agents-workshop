

# Troubleshooting

## Authentication issue
If you are not authenticated you will get a 401 Unauthorized error when you try to send a message. Make sure you have set the `ANTHROPIC_API_KEY` environment variable with your API key.
```bash
Chat with Claude (use 'ctrl-c' to quit)
You: what can you do
Error: POST "https://api.anthropic.com/v1/messages": 401 Unauthorized (Request-ID: req_011CYv2f6aZdYoeR1acx8dKL) {"type":"error","error":{"type":"authentication_error","message":"x-api-key header is required"},"request_id":"req_011CYv2f6aZdYoeR1acx8dKL"}
```

