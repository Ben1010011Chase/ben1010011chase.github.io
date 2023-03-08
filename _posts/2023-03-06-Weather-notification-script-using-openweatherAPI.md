---
title: "Weather Notification Script Using OpenWeather API"
date: 2023-03-04
categories: blog
tags: [python, smtp, ssl/tls]
---

**Problem:** My apartment, like many others, has a snow removal policy which requires renters to move their cars from the parking lot after a snowfall or risk getting towed. **Solution:** I created a simple weather notification script using Python and [OpenWeather API](https://openweathermap.org/api).

I split the script into three files: `snow.py` contains the main script that retrieves weather data and triggers notifications. `send_email.py` contains a function that sends email or text notifications via [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol). Sensitive information is kept in a YAML config file that looks something like this:

```yaml
SMTP_SERVER: smtp.office365.com
SMTP_PORT: 587
EMAIL_ADDRESS: #email
EMAIL_PASSWORD: #pass

BEN_CELL: #cell1
JENIKA_CELL: #cell2

WEATHER_API_KEY: #api_key
LATITUDE: #lat 
LONGITUDE: #lon
```

Then the Python [`YAML`](https://pyyaml.org) library can be used to read the config file and parse it into a dictionary. The following line of code reads the contents of the config.yml file, parses it into a dictionary using the yaml.safe_load function, and assigns the resulting dictionary to the config variable.

```python
with open('email_config.yaml', 'r') as f:
    config = yaml.safe_load(f)
```

Here, we're opening the config.yml file in read mode using the built-in `open` function. We're also using a `with` statement to ensure that the file is properly closed after we're done with it, even if an exception is raised.

The `yaml.safe_load` function is used to read the contents of the file and parse it into a Python dictionary. This function is a part of the PyYAML library and is designed to safely load YAML data without executing any arbitrary code. It is [recommended](https://security.openstack.org/guidelines/dg_avoid-dangerous-input-parsing-libraries.html) to use `yaml.safe_load` instead of `yaml.load`, since the latter can be vulnerable to code injection attacks.

## The `snow.py` file

The goal of this script is to send notifications if it's going to snow more than a certain threshold overnight. The script uses the OpenWeather API to retrieve forecast weather data for a specific location (specified by latitude and longitude in a YAML config file). The script then extracts data for the overnight period (from 6pm to 9am) and calculates the snow accumulation (if any) from that data.

Here's the API call:

```python
url = ('https://api.openweathermap.org/data/2.5/'
        f'forecast?lat={lat}&lon={lon}&units=metric'
        f'&cnt=5&appid={weather_api_key}')
response = requests.get(url)
```

The `cnt` parameter limits the number of forecast results returned by the API to 5. This means that the API will only return data for the next five forecast periods (in three hour increments), rather than returning data for all forecast periods.

Limiting the amount of data returned by an API can be beneficial for several reasons. First, it can help to reduce the load on the server, since the server does not need to generate and transmit as much data for each API call. This can improve the performance and scalability of the API, especially for high-traffic applications. Second, limiting the amount of data returned by an API can reduce the amount of data that needs to be transferred over the network. This can be particularly important for mobile or low-bandwidth applications, where network connectivity may be limited or unreliable. For small GET requests like this, it's probably not necessary to limit the amount of data returned by the API. However, it's still a good idea to limit the amount of data returned by an API whenever possible.

Next, parse the response JSON to get the weather data

```python 
weather_data = response.json()
```

and extract the total snow accumulation for the overnight period (from 6pm to 9am).

```python
now = datetime.now()
start_time = now.replace(hour=18, minute=0)
end_time = start_time + timedelta(hours=15)

overnight_data = [data for data in weather_data['list'] 
                  if start_time <= datetime.fromisoformat(data['dt_txt']) < end_time]

snow_accumulation = sum(data['snow']['3h'] if 'snow' in data and
                        '3h' in data['snow'] else 0 for data in
                         overnight_data)
```                           

[//]: # (Representation errors in snow_accumulation https://en.wikipedia.org/wiki/Round-off_error https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)

## The `send_mail.py` file

If the snow accumulation is above a certain threshold, `snow.py` displays a system notification and sends an email or text to multiple recipients (specified in the config file). If the script encounters an error while retrieving weather data or sending an email, it also sends a text to a specified recipient. Here is `send_mail.py`:

```python   
import smtplib
import yaml

def send_email(subject, body, *to_address):
    with open('config.yml', 'r') as f:
        config = yaml.safe_load(f)
    smtp_server = config['SMTP_SERVER']
    smtp_port = config['SMTP_PORT']
    email_address = config['EMAIL_ADDRESS']
    email_password = config['EMAIL_PASSWORD']

    message = (f'From: {email_address}\nTo:{", ".join(to_address)}'
                f'\nSubject: {subject}\n\n{body}')

    try:
        with smtplib.SMTP(smtp_server, smtp_port) as server:
            server.ehlo()
            server.starttls()
            server.login(email_address, email_password)
            server.sendmail(email_address, to_address, message)
            server.quit()
    except smtplib.SMTPAuthenticationError:
        print("Error: Could not authenticate with SMTP server.")
    except smtplib.SMTPException as e:
        print(f"Error: {e}")
```

The Python SMTP library comes with two methods for creating a secure SMTP connection: `SMTP_SSL()` and `.starttls()`.
The decision to use `SMTP_SSL()` or `.starttls()` to secure your SMTP connection depends on your email server's configuration and your application's requirements. `SMTP_SSL()` creates a secure SSL/TLS-encrypted connection to your email server right from the start. On the other hand, `.starttls()` starts an unencrypted SMTP connection, then sends a STARTTLS command to initiate the TLS handshake and upgrade the connection to a secure TLS-encrypted connection.

Some email servers may not support `SMTP_SSL()` and require the use of `.starttls()`. Furthermore, some email servers may require the use of specific encryption protocols or SSL certificates, which can be specified in the context parameter of the `.starttls()` method. Typically email servers will use SMTP port 465 for `SMTP_SSL()` and SMTP port 587 for `.starttls()`.

In general, if your email server supports both `SMTP_SSL()` and `.starttls()`, and your application does not have any specific requirements for one method over the other, it is recommended to use `SMTP_SSL()` for better security and performance.

[//]: # (add reference to above paragraph) 

## Conclusion

Overall, this script provides a convenient way to stay informed about snow conditions and can be easily customized to suit different locations and weather thresholds. It can also be scheduled to run automatically using a task scheduler or a cron job, ensuring that the user is always up-to-date on the weather conditions. OpenWeather also has a new [solar radiation API](https://openweathermap.org/api/solar-radiation). 


