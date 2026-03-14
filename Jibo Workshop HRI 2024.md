
### Resources:
https://github.com/jibodev/ShofEL2-for-T124-Jibo/tree/t124 - Github Repo
https://hri2024.jibo.media.mit.edu/Setting-Up-Your-Codespace - Guide to setup a repo

- - -
### About:
From the jibo workshop i have discovered that they have actually included a lot of info on how jibo expects the server to give it data, and by that i dont mean that it uses [[ESML]], but what is available to use in every response!. Furthermore we also have access to some "examples" through the git repo on how to setup a [[Robot Os|ROS]] container, and I was able to extract the source code from the scratch extension over at [Scratch Playground](https://playground.raise.mit.edu/firebase-scratch/).

- - -


Were gonna have to analyse the stuff to go further one , and as i was looking at the files i discovered a file named `jibo_chatGPT.ipynb` which sparked some attention, and it appears that they have made a honest attempt at trying to hook it up to chat gpt

![[JiboChatGPT.png]]

And there is a section where they clearly tried to make it act like jibo

```python

def main(args=None):

    teleop_connection = JiboTeleop()

    time.sleep(2)

  

    system_message =  "You are Jibo, a friendly social robot who likes to chat about different topics." \

                      "Pick a random topic to discuss and use personalized stories." \

                      "Always respond in one short fun and exciting sentence, " \

                      "and use language that is easy to understand for all ages." \

                      " Give turns to user to respond."

    # Spin in a separate thread

    thread = threading.Thread(target=rclpy.spin, args=(teleop_connection, ), daemon=True)

    thread.start()

...

```


