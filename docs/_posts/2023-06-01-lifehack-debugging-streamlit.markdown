---
layout: post
title:  "Life Hack: Debugging Streamlit Apps in VS Code"
date:   2023-06-01 16:55:50 +0100
category: Work
tags: LifeHacks
---
Have you ever tried to debug your Streamlit app like a normal python script? In case you want to see how it's done in VS Code, continue reading this brief step-by-step guide. 
<!--more-->

Actually, you only need to create a small config file in your project folder. VS Code then automatically retrieves the necessary debugger settings from it.
  
1. Click on the debugger icon in the sidebar on the left
2. Click on "Create launch.json"
3. Edit the variables as shown below
  
Here's how my launch file looks like:
  
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "debug streamlit",
            "type": "python",
            "request": "launch",
            "program": "c:\\Users\\<Username>\\Python\\<environment name>\\Scripts\\streamlit.exe",
            "args": [
                "run",
                "app.py"
            ],
            "console": "integratedTerminal",
            "cwd": "${fileDirname}"
        }
    ]
}
```
  
Just replace the path in the `"program"` variable with your own streamlit path. You can probably tell from the double backslashes that I'm using VS Code on Windows. `"${fileDirname}"` is a dynamic working directory, which takes advantage of the internally defined fileDirname variable in VS Code. It lets the debugger find the right working directory for the run.

Hope this little life hack will do the trick for you!
