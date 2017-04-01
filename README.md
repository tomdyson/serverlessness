# Serverlessness

## Frameworks

- AWS Lambda
- Google Cloud Functions
- Azure Functions
- Serverless Framework
- Chalice
- Zappa

## Schedule scraper app.py

```python
import datetime
import urllib
from chalice import Chalice
from bs4 import BeautifulSoup

SCHEDULE_URL = 'http://tom.s3.amazonaws.com/schedule.html'
CONFERENCE_DAYS = {
    'monday': 'April 3, 2017',
    'tuesday': 'April 4, 2017',
    'wednesday': 'April 5, 2017',
    'thursday': 'April 6, 2017',
    'friday': 'April 7, 2017',
}

def parse_schedule(html):
    soup = BeautifulSoup(html, "html.parser")
    schedule = {}
    for day in soup.find_all('section', class_='schedule-day'):
        date, location = day.find('h3').text.split(' @ ')
        talks = []
        for talk in day.find_all('article', class_='talk'):
            time = talk.find_all(class_='talk__time')[0].text
            title_node = talk.find(class_='talk__title')
            title = title_node.text
            if "Coffee" in title:
                title += " :coffee:"
            elif "Lunch" in title:
                title += " :knife_fork_plate:"
            url = title_node.attrs.get('href')
            speaker_node = talk.find(class_='talk__author')
            speaker = speaker_node.text if speaker_node else ''
            talks.append([time, title, speaker, url])
        schedule[date] = {}
        schedule[date]['location'] = location
        schedule[date]['talks'] = talks
    return schedule

app = Chalice(app_name='djangoconeu')
app.debug = True

@app.route('/')
def index():
    html = urllib.urlopen(SCHEDULE_URL).read()
    schedule = parse_schedule(html)
    requested_day = app.current_request.query_params.get('text').lower().strip()
    today = datetime.datetime.now().strftime('%B %-d, %Y')
    if len(requested_day):
        date = CONFERENCE_DAYS.get(requested_day, today)
    else:
        date = today
        date = 'April 3, 2017'  # TODO: delete me
    today = schedule.get(date)
    user = app.current_request.query_params.get('user_name')
    text = "Hello %s! :wave:\n" % user if user else ""
    text += "Here's the schedule for %s, in %s:\n" % (date, today['location'])
    for (time, title, speaker, url) in today['talks']:
        speaker_fmt = '(%s)' % speaker if speaker else ''
        text += ' - %s <https://2017.djangocon.eu%s|%s> %s\n' % (time, url, title, speaker_fmt)
    return {'text': text}

@app.route('/all', cors=True)
def all_events():
    html = urllib.urlopen(SCHEDULE_URL).read()
    schedule = parse_schedule(html)
    return {'schedule': schedule}
```
