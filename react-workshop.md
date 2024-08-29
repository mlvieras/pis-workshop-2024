# React Workshop

## Setting up the project

### Install the appropriate versions of Node and NPM

There's a multitude of ways to install and manage Node versions. Usually, projects will require you to be able to spin up many versions of Node. It's simply too annoying to uninstall and install a different version each time. That's why there exist different version managers that support Node. Some of them are `nodenv`, `nvm` and `asdf`.

At Xmartlabs we've mostly used [nodenv](https://github.com/nodenv/nodenv), so these next instructions take for granted that you have it installed on your machine.

```sh
nodenv install 20.8.1
npm i -g npm@10.8.2
```

### Cloning the template

First step is cloning Xmartlabs' Create React App template.

* Go to [the Github repo](https://github.com/xmartlabs/react-template-xmartlabs)
* Run `degit`: `npx degit@latest xmartlabs/react-template-xmartlabs frontend`

### Run the newly-created project

```sh
cd frontend
npm install
npm start
```

## Let's start doing stuff!

We're going to create a very small application that lists some tasks the user has to do.

### No backend? No worries!

In order to quickly have a "backend" running, we're going to use [`json-server`](https://github.com/typicode/json-server).

First, we need to create a `db.json` file on the root of the react workshop:

```sh
touch db.json
```

Now, add sample data inside:

```json
{
  "messages": [
    {
      "id": 1,
      "content": "Don't forget: wash your clothes",
      "due_date": "2022-08-19T18:38:58.278Z",
      "is_complete": false
    },
    {
      "id": 2,
      "content": "Study for my exam",
      "due_date": "2022-08-20T10:00:58.278Z",
      "is_complete": true
    },
    {
      "id": 3,
      "content": "Take out the trash",
      "due_date": "2022-05-03T09:00:00.278Z",
      "is_complete": false
    },
    {
      "id": 4,
      "content": "Take out the trash",
      "due_date": "2022-03-01T18:38:58.278Z",
      "is_complete": false
    }
  ]
}
```

Let's now start our server:

```sh
npx json-server --watch db.json -p 3001
```

It should output something like:

```text
  \{^_^}/ hi!

  Loading db.json
  Done

  Resources
  http://localhost:3001/messages

  Home
  http://localhost:3001

  Type s + enter at any time to create a snapshot of the database
  Watching...
```

### Fetching the messages

We'll need to create two things:

* The controller on the frontend that will handle the request.
* The serializer that will process the data.

Here's the controller:

```ts
// src/networking/controllers/message-controller.ts

import { MessageSerializer } from 'networking/serializers/message-serializer';
import { ApiService } from 'networking/api-service';
import { API_ROUTES } from 'networking/api-routes';

class MessageController {
  static async getMessages() : Promise<Message[]> {
    const response = await ApiService.get<RawMessage[]>(API_ROUTES.MESSAGES);
    return (response[]).map(MessageSerializer.deSerialize);
  }
}

export { MessageController };
```

The serializer:

```ts
// src/networking/serializers/message-serializer.ts

class MessageSerializer {
  static deSerialize(data: RawMessage) : Message {
    return {
      id: data.id,
      content: data.content,
      dueDate: new Date(data.due_date),
      isComplete: data.is_complete,
    };
  }
}

export { MessageSerializer };
```

The types:

```ts
// src/networking/types/message.d.ts

type RawMessage = {
  id: number,
  content: string,
  due_date: string,
  is_complete: boolean,
};

type Message = {
  id: number,
  content: string,
  dueDate: Date,
  isComplete: boolean,
};
```

Then, we need to add the path to our endpoint:

```ts
// src/networking/api-routes.ts

const API_ROUTES = {
    MESSAGES: "/messages",
};
```

And lastly, update our enviroment variables. We need to rename the .env.development.local.example to .env.development.local
and replace the VITE_API_BASE_URL with the URL of our endpoint (you can see where the json-server is running in the terminal).

### Fetch our data!

To achieve this, we need to modify our home componenet slightly.
We'll replace the content in our home for these lines:

```ts
// src/pages/homes/home.tsx
import { classnames } from "helpers/utils";
import globalStyles from "assets/stylesheets/global-styles.module.scss";
import { useEffect, useState } from "react";
import { MessageController } from "networking/controllers/message-controller";
import { Button } from "common/button";
import { TextField } from "common/text-field";
import styles from "./home.module.scss";

const Home = () => {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
      const fetch = async () => {
        const response = await MessageController.getMessages();
        setMessages(response);
      };
      fetch();
    }, []);

  return (
    <div
      className={classnames(styles.container, globalStyles.genericContainer)}
    >
      <div className={styles.subContainer}>
        <div>
          <h2>Messages</h2>
          {messages.map((m) => (
            <div
              key={m.id}
              style={{
                flexDirection: "row",
                display: "flex",
                justifyContent: "space-between",
              }}
            >
              <div>{`[${m.isComplete ? "✅" : "❌"}] ${m.content}`}</div>
              <div>{m.dueDate.toLocaleDateString('US')}</div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export { Home };

```

If we have succeeded, we should see the messages in our home.

### Put new data!

Now, in our final step of this workshop, we'll add a form to create messages. We'll need to add a new endpoint
to our controller and some logic in our home.


In the controller:

```ts
// src/networking/controllers/message-controller.ts

static async createMessage(message: Message): Promise<Message> {
  const response = await ApiService.post<RawMessage>(API_ROUTES.MESSAGES, {
    body: JSON.stringify({
      content: message.content,
      due_date: new Date(message.dueDate),
      is_complete: message.isComplete,
    }),
  });

  if (response) {
    return MessageSerializer.deSerialize(response);
  }
  return {} as Message;
}
```

In home:

```ts
// src/pages/homes/home.tsx

// we need add this below the other useState
const [content, setContent] = useState<string | undefined>(undefined);
const [dueDate, setDueDate] = useState<string | undefined>(undefined);
const [isComplete, setIsComplete] = useState(false);
const [idLastMessage, setIdLastMessage] = useState<string>("");


// under useEffect add:
const validValues = content && dueDate;

const handleSubmit = async (e: any) => {
  e.preventDefault();

  if (idLastMessage && content && dueDate) {
    const newMessage = await MessageController.createMessage({
      id: idLastMessage.toString(),
      content,
      dueDate: new Date(dueDate),
      isComplete,
    });

    setIdLastMessage(idLastMessage + 1);
    setMessages((prevState) => [...prevState, newMessage]);

    setContent("");
    setIsComplete(false);
  }
};

//and below the Messages' div add:
<div className={styles.formContainer}>
  <form onSubmit={handleSubmit} className={styles.form}>
    <TextField
      label="Id"
      name="id"
      onChange={(event) => {
        setIdLastMessage(event.target.value);
      }}
      value={idLastMessage}
    />
    <TextField
      label="Content"
      name="content"
      onChange={(event) => {
        setContent(event.target.value);
      }}
      value={content}
    />
    <div>
      <label htmlFor="due_date">Due date:</label>
      <input
        type="date"
        id="due_date"
        name="due_date"
        value={dueDate}
        onChange={(event) => {
          setDueDate(event.target.value);
        }}
      />
    </div>
    <div>
      <label htmlFor="is_complete">Is completed?</label>
      <input
        type="checkbox"
        id="is_complete"
        name="is_complete"
        checked={isComplete}
        onChange={() => {
          setIsComplete((prevState) => !prevState);
        }}
      />
    </div>
    <Button type="submit" disabled={!validValues}>
      Save message
    </Button>
  </form>
</div>
```
If we succeeded, we should be able to add a message, and the list should automatically update with the new message.

Good Job!

Note: we strongly recommend read [`useState`](https://react.dev/reference/react/useState), [`useEffect`](https://react.dev/reference/react/useEffect) to understand the workshop.
