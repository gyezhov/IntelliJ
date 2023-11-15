import os
import splunklib.client as client
import splunklib.results as results
from slack import WebClient
from slack.errors import SlackApiError

# Fetch credentials from environment variables
splunk_host = os.getenv("SPLUNK_HOST")
splunk_username = os.getenv("SPLUNK_USERNAME")
splunk_password = os.getenv("SPLUNK_PASSWORD")
search_query = "index=artemix | head 100"

slack_webhook_url = os.getenv("SLACK_WEBHOOK_URL")
slack_channel = "#your-channel"

if not splunk_host or not splunk_username or not splunk_password or not slack_webhook_url:
    print("Please set environment variables for credentials.")
    exit(1)

# Initialize the Splunk service and log in
service = client.Service(splunk_host, username=splunk_username, password=splunk_password)

try:
    # Run the search query and get the job
    job = service.jobs.create(search_query)

    # Create a message for Slack
    slack_message = "Search Job Results:\n"

    # Stream the job results and process as they become available
    for event in results.ResultsReader(job.results()):
        # Extract relevant job details
        if 'eventCount' in event:
            events_scanned = event['eventCount']
            slack_message += f"Events Scanned: {events_scanned}\n"

        if 'runDuration' in event:
            execution_time = event['runDuration']
            execution_time_seconds = "{:.3f}".format(execution_time / 1000)
            slack_message += f"Execution Time (seconds): {execution_time_seconds}\n"

    # Post the message to Slack
    slack_client = WebClient(token=slack_webhook_url)
    response = slack_client.chat_postMessage(channel=slack_channel, text=slack_message)

    # Check if the message was sent successfully
    if response['ok']:
        print("Message sent to Slack successfully.")
    else:
        print("Failed to send message to Slack.")

except Exception as e:
    print(f"An error occurred: {str(e)}")

# Cleanup: Close the Splunk service
service.logout()
