---
layout: post
title: "Telegram bots in NodeJS"
author: Javier Garcia
description: "Touching base with Telegram's bot framework."
category: bots
tags: bots, telegram, nodejs, automation
---

Recently I started exploring Telegram's Bot API out of mere curiosity to eventually decide to make a couple of bots in NodeJS. Suprisingly found it was easy and fun, so here I am sharing it with you.

To be honest, I chose Node because it's what I'm enjoying the most at the moment, but after googling a bit I figured the same bots I'll show you today can be done with Python, Java or any other language just the same. There are multiple bot frameworks developed by the community that aid us in this and make it easy as drinking a beer!

In this post I'll just explain a bit about the more complex bot I developed, **WeatherWarnBot**. It's a bot that fetches a weather API both both on demand and proactively to warn us when it's going to rain on the following day.

First I designed a small **command API** for the users to interact with my bot. The commands are:
1. `/help`, which returns all the availiable commands
2. `/forecast`, which gives us the following day's weather of a city we can input by parameter
3. `/schedule` and `/unschedule`, for the proactive notifications, also with an input parameter for the city.

### Frameworks
Once I knew what I wanted the bot to do, I just had to figure out how to do it. After googling for a while, I came across a framework for bots developed in NodeJS, [Telegraf](http://http://telegraf.js.org).

Telegraf is a small Telegram API wrapper, very minimalistic, which also includes some things of its own. In this case, I used a Telegraf high-level wrapper, [micro-bot](https://github.com/telegraf/micro-bot), which makes making small bots even easier.

### Project Structure
Here is how I decided to design my bot:

```
weatherWarnBot
 |_helpers
 |  |_cron-scheduler.js
 |  |_templates.js
 |  |_weather.js
 |_bot.js
 |_config.js
```

On one folder I have the helper files which will provide me the logic to perform the weather API callout, the cron-scheduling and the message templates, and in the outer folder I'll just have the proper bot with it's config file with tokens, etc.

The bot makes the same API call to OpenWeatherMap both for the on-demand and the scheduled features. The difference relies on the fact that for in the scheduled mode we simply check the API's response's weather code to check if it's rainy or not.

Regarding the scheduling, what the bot does is collect the user's schedule request, create a cron job and save it in a map so that if the user wants at some point to unschedule that job, he can access it. At the moment, I'm managing all records in that map, but at some point that should go into a DB so it can persist and not loose the jobs everytime the bot is turned off.

### Implementation
*Note: here is a shortened version of the bot for easier understanding. It only contains the on-demand weather forecast. If you're interested in seeing the full version, with the scheduling feature just visit my [github repo](http://github.com/Manzanit0/simple-tgram-bots) where you'll find the code and some quick documentation.*


In this project, the `bot.js` file orquestrates the whole thing. It implements the `micro-bot` package and registers commands I mentioned earlier. In this snippet we just have the command for the forcasting and help, but in the full version which you can find in github there is also an implementation of schedule/unschedule commands.

Since we will be listening for a command such as `/forecast Madrid ES`, or something of the kind, we will use RegEx to recognize the parameters. I won't be breaking down the regular expression in this post, but, in a nutshell, it simply matches the first two words following the command and interprets the first as the city and second as the country ISO code, which we then use to call the helper methods.

Do notice that we are using `async/await` functions to force the code to act synchronously so the message is never sent before we have the response from the weather API. Apart from that, the code is very simple.

###### bot.js
```javascript
'use strict';

// Telegraf based framework.
const { Composer } = require('micro-bot');

const bot = new Composer();

bot.hears(/\/forecast (\S+) (\S+)/, async ({ match, reply }) =>
    reply(await weather.getForecastMessage(match[1], match[2], weather.daysEnum.TOMORROW)));

bot.hears(/\/help/, ({ reply }) =>
    reply(templates.help));

// Export bot handler
module.exports = bot;
```

Next is the config file with all our _sensitive_ information. In this case, all I need to save is my API key to access the OpenWeatherMap API, but if we were to access other APIs, we could have it also here. 

In a real-life scenario, we would never save our API keys in a file which we upload to a repository since its not safe, but for this project I decided to do it for mere simplicity. Most of the times, if you use for example Heroku, OpenShift or any other platform of the kind, you can configure the keys as system variables.

###### config.js
```javascript
'use strict';

const weatherApiKey = '<SomeKey>';

module.exports.weatherApiKey = weatherApiKey;
```

Next is just a common helper file which contains all the logic regarding the callouts to the weather API and the templates which contains all the text for the messages that the bot will send us. 

Regarding the weather API, I chose [OpenWeatherMap](https://openweathermap.org) because it was free and fairly good, but I am aware that Yahoo also has a pretty good API for weather stuff.

As for the templates, they're in Spanish simply because I'm Spanish :)

###### helpers/weather.js
```javascript
'use strict';

const req = require('req');

const config = require('../config'),
    templates = require('./templates');

const daysEnum = {
    TODAY: 0,
    TOMORROW: 1,
    IN_2_DAYS: 2,
    IN_3_DAYS: 3,
    IN_4_DAYS: 4,
    IN_5_DAYS: 5,
    IN_6_DAYS: 6,
    IN_7_DAYS: 7,
    IN_8_DAYS: 8
};

const callAPI = async (city, countryCode) => {
    const api = `http://api.openweathermap.org/data/2.5/forecast/daily/?q=${city},${countryCode}&APPID=${config.weatherApiKey}&units=metric&lang=es`;
    const body = await req(api);
    return JSON.parse(body);
};

const getForecast = async (city, countryCode, day) => {
    const result = await callAPI(city, countryCode);
    return day === null ? result.list[daysEnum.TOMORROW] : result.list[day];
};

const getForecastMessage = async (city, countryCode, day) => {
    const forecast = await getForecast(city, countryCode, day);
    let message;
    try {
        message = templates.weatherReport(forecast, city);
    } catch (err) {
        message = templates.errorMessage;
    }

    return message;
};

module.exports = { daysEnum, getForecast, getForecastMessage };
```

###### helpers/templates.js
```javascript
'use strict';
const UNIX_DT_TRANSFORM_RATIO = 1000;

const help =
    `
    Los siguientes comandos están disponibles para su uso: \n
    ✔️ /forecast {CIUDAD} {CODIGO_PAIS}
    \t Devuelve el tiempo para ciudad en estos momentos.\n
    ✔️ /schedule {CIUDAD} {CODIGO_PAIS}
    \t Me programo para avisaros el día antes en caso de que vaya a llover,haya tormenta o cambios de temperatura bruscos.\n
    ✔️ /unschedule {CIUDAD} {CODIGO_PAIS}
    \t Me desprogramo para que no lleguen más notificaciones en el futuro.
    `;

const weatherReport = (forecast, cityName) =>
    `🚩 ${cityName}
    - - - - - - - - - - - - - - - - - - - - - -
    🕘 ${new Date(forecast.dt * UNIX_DT_TRANSFORM_RATIO).toDateString()}
    🌀 ${forecast.weather[0].description}
    🔰 ${forecast.temp.min}°C - ${forecast.temp.max}ºC
    💧 ${forecast.humidity}%
    💨 ${forecast.speed} m/s
    - - - - - - - - - - - - - - - - - - - - - -
    `;

const errorMessage = 'Ha habido un problema contactando con la API del tiempo. Para más información ponte en contacto con mi desarrollador 😬';

module.exports = { errorMessage, help, weatherReport };
```

Now, putting together all these pieces we have a small, simple yet functional Telegram bot which helps us to check the weather. As you have maybe imagined, the possibilities are practicably limitless. We could use Telegram bots to check for lyrics, compile code, control our calendar or even IoT stuff like control our home's thermostat!

In any case, may you make many bots and happy coding! :)
