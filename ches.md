# Using the BC Gov "CHES" email system in shell scripts

Summary: Use the CHES email system to improve email delivery from automated jobs and processes.

## Overview
System administrators and application developers often find it helpful to receive email messages from automated jobs with information about the job outcome, system status, etc.  Historically, it has generally been possible to send email directly from a server using a single command in a shell script, which is very convenient.  Now, however, the BC Gov email servers block such emails.  One solution is to send the emails through the CHES email system.

https://getok.nrs.gov.bc.ca/app/about

## Request a CHES account
In order to request an account in the CHES system, you will need to provide a unique acronym for the application in question and a brief description of the application and the ministry that manages it.

https://getok.nrs.gov.bc.ca/app/requestAccount

Each account will include a client ID and separate passwords for each of the Dev, Test, and Prod environments.

Record the ID, password, and token URL for each environment.

## Using the CHES service account
Interactions with the email system are via API calls.  You will have to:
- Request a token from CHES
- Format the message body
- Submit the token, other headers, and message body to the API

### Fetch a token
Assign the CHES user ID and password to environment variables in your script, such as:
- CHES_USER_ID
- CHES_PASSWORD

To make the access token request, we add these headers:
- `Content-Type: application/x-www-form-urlencoded`
- `grant_type=client_credentials`

and supply the username and password.

Here's a basic example, but we'll look at a complete example below.

`curl -k -H "Content-Type: application/x-www-form-urlencoded" -d "grant_type=client_credentials" --user "${CHES_USER_ID}:${CHES_PASSWORD}" ${TOKEN_URL}`

In your script, it's helpful to know that the token request was successful before proceeding, so we'll capture the HTTP resonse code and also write the token out to a file.  In this example:
- `-w "%{http_code}"` tells curl to print the response code to STDOUT
- `-s` enables "silent mode", so that the normal output is not printed to STDOUT
- `-o ${TOKEN_FILE}` tells curl to write the token response itself to the file path that has been assigned to TOKEN_FILE

`curl -k -H "Content-Type: application/x-www-form-urlencoded" -d "grant_type=client_credentials" --user "${CHES_USER_ID}:${CHES_PASSWORD}" -s -o ${TOKEN_FILE} -w "%{http_code}" ${TOKEN_URL}`

We check the HTTP response.  If it's a `200`, we'll extract the access token from the response that we received.
```
# Request a token for CHES and verify the response code
# -----------------------------------------------------
if [ -f "$TOKEN_FILE" ]; then rm $TOKEN_FILE; fi
TOKEN_RESPONSE=`curl -k -H "Content-Type: application/x-www-form-urlencoded" -d "grant_type=client_credentials" --user "${CHES_USER_ID}:${CHES_PASSWORD}" -s -o ${TOKEN_FILE} -w "%{http_code}" ${TOKEN_URL}`
if [[ "$TOKEN_RESPONSE" == "200" ]]; then
  CHES_TOKEN=`cat ${TOKEN_FILE} | sed -r 's/^.*access_token":"([^"]+).*$/\1/'`
else
  echo "Error getting access token from CHES"
  exit 1
fi
```

### Prepare message content
The message body and other email fields must be set in a JSON string, which can be tricky.  We'll probably be using shell variables in the command, so we must wrap the curl arguments in double quotes (as opposed to single quotes), but the JSON string itself must contain double quotes, so those will have to be escaped.  And if you have a multiline block of text for your message body it gets trickier, because the newlines will have to be encoded, too.

We can achieve this by piping our block of text to `sed` for two different transformations and then writing the output to a file.  We'll then assign the contents of the file to a variable for use in the API call to CHES.
```
# Prepare the message body
# sed:
#   :a create a label 'a'
#   N append the next line to the pattern space
#   $! if not the last line, ba branch (go to) label 'a'
#   s substitute, /\n/ regex for new line, replace with escaped newline, /g global match
#   https://stackoverflow.com/questions/1251999/how-can-i-replace-each-newline-n-with-a-space-using-sed
# then encode double quotes
# ------------------------------------------------------------------------------
echo "$MESSAGE_BODY" | sed ':a;N;$!ba;s/\n/\\n/g' | sed 's/"/\\u0022/g' > ${MESSAGE_BODY_FILE}
EMAIL_BODY=`cat ${MESSAGE_BODY_FILE}`
```

### Send the message by calling the CHES API
We send the email by calling the API with the following arguments:
- `-H "Content-Type: application/json"` indicates we are submitting data in JSON format
- `-H "Authorization: Bearer ${CHES_TOKEN}"` this is the access token we received earlier
- `-d "{\"bodyType\": \"text\", \"body\": \"${EMAIL_BODY}\", \"from\": \"my-app-job@gov.bc.ca\", \"subject\": \"my app job\", \"to\": [\"${EMAIL_RECIPIENT}\"]}"` all the fields needed for the email

All together, we have:

`curl -k -H "Content-Type: application/json" -H "Authorization: Bearer ${CHES_TOKEN}" -d "{\"bodyType\": \"text\", \"body\": \"${EMAIL_BODY}\", \"from\": \"my-app-job@gov.bc.ca\", \"subject\": \"my app job\", \"to\": [\"${EMAIL_RECIPIENT}\"]}" ${CHES_URL}`

## Summary
The process itself is quite simple:
- fetch access token
- format email body
- send email

but the formatting can be tricky.  Here's an example that uses the three elements demonstrated above.

```
# Request a token for CHES and verify the response code
# -----------------------------------------------------
if [ -f "$TOKEN_FILE" ]; then rm $TOKEN_FILE; fi
TOKEN_RESPONSE=`curl -k -H "Content-Type: application/x-www-form-urlencoded" -d "grant_type=client_credentials" --user "${CHES_USER_ID}:${CHES_PASSWORD}" -s -o ${TOKEN_FILE} -w "%{http_code}" ${TOKEN_URL}`
if [[ "$TOKEN_RESPONSE" == "200" ]]; then
  CHES_TOKEN=`cat ${TOKEN_FILE} | sed -r 's/^.*access_token":"([^"]+).*$/\1/'`
else
  echo "Error getting access token from CHES"
  exit 1
fi

# Prepare the message body
# sed:
#   :a create a label 'a'
#   N append the next line to the pattern space
#   $! if not the last line, ba branch (go to) label 'a'
#   s substitute, /\n/ regex for new line, replace with escaped newline, /g global match
#   https://stackoverflow.com/questions/1251999/how-can-i-replace-each-newline-n-with-a-space-using-sed
# then encode double quotes
# ------------------------------------------------------------------------------
echo "$MESSAGE_BODY" | sed ':a;N;$!ba;s/\n/\\n/g' | sed 's/"/\\u0022/g' > ${MESSAGE_BODY_FILE}
EMAIL_BODY=`cat ${MESSAGE_BODY_FILE}`

# Send the email by calling the CHES API
# --------------------------------------
curl -k -H "Content-Type: application/json" -H "Authorization: Bearer ${CHES_TOKEN}" -d "{\"bodyType\": \"text\", \"body\": \"${EMAIL_BODY}\", \"from\": \"my-app-job@gov.bc.ca\", \"subject\": \"my app job\", \"to\": [\"${EMAIL_RECIPIENT}\"]}" ${CHES_URL}
```

## Helpful links
CHES service page: https://getok.nrs.gov.bc.ca/app/about

Request an account: https://getok.nrs.gov.bc.ca/app/requestAccount

CHES API info: https://ches.nrs.gov.bc.ca/api/v1/docs

sed, a stream editor: https://www.gnu.org/software/sed/manual/sed.html


