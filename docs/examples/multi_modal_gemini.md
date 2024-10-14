---
title: Utilizing Gemini for Multi-Modal Data Processing with Audio Files
description: Learn how to use Gemini with Google Generative AI to process audio files efficiently in multi-modal applications.
---

# Using Gemini with Multi Modal Data

This tutorial shows how to use `instructor` with `google-generativeai` to work with multi-modal data. In this example, we'll demonstrate three ways to work with audio files.

We'll be using this [recording](https://storage.googleapis.com/generativeai-downloads/data/State_of_the_Union_Address_30_January_1961.mp3) that's taken from the [Google Generative AI cookbook](https://github.com/google-gemini/cookbook/blob/main/quickstarts/Audio.ipynb).

## Normal Message

The first way to work with audio files is to upload the entire audio file and pass it into the LLM as a normal message. This is the easiest way to get started and doesn't require any special setup.

```python
import instructor
import google.generativeai as genai
from pydantic import BaseModel


client = instructor.from_gemini(
    client=genai.GenerativeModel(
        model_name="models/gemini-1.5-flash-latest",
    ),
    mode=instructor.Mode.GEMINI_JSON,  # (1)!
)

mp3_file = genai.upload_file("./sample.mp3")  # (2)!


class Description(BaseModel):
    description: str


resp = client.create(
    response_model=Description,
    messages=[
        {
            "role": "user",
            "content": "Summarize what's happening in this audio file and who the main speaker is",
        },
        {
            "role": "user",
            "content": mp3_file,  # (3)!
        },
    ],
)

print(resp)
#> description="The main speaker is President John F. Kennedy, and he's giving a
#> State of the Union address to a joint session of Congress. He begins by
#> acknowledging his fondness for the House of Representatives and his long
#> history with it. He then goes on to discuss the state of the economy,
#> highlighting the difficulties faced by Americans, such as unemployment and
#> low farm incomes. He also touches on the Cold War and the international
#> balance of payments. He speaks of the need to strengthen the US military,
#> and he also discusses the importance of international cooperation and the
#> need to address global issues like hunger and illiteracy. He ends by urging
#> his audience to work together to face the challenges that lie ahead."
```

1. Make sure to set the mode to `GEMINI_JSON`, this is important because Tool Calling doesn't work with multi-modal inputs.
2. Use `genai.upload_file` to upload your file. If you've already uploaded the file, you can get it by using `genai.get_file`
3. Pass in the file object as any normal user message

## Inline Audio Segment

!!! note "Maximum File Size"

    When uploading and working with audio, there is a maximum file size that we can upload to the api as an inline segment. You'll know when this error is thrown below.

    ```
    google.api_core.exceptions.InvalidArgument: 400 Request payload size exceeds the limit: 20971520 bytes. Please upload your files with the File API instead.`f = genai.upload_file(path); m.generate_content(['tell me about this file:', f])`
    ```

    When it comes to video files, we recommend using the file.upload method as shown in the example above.

Secondly, we can also pass in a audio segment as a normal message as an inline object as shown below. This requires you to install the `pydub` library in order to do so.

```python
import instructor
import google.generativeai as genai
from pydantic import BaseModel
from pydub import AudioSegment

client = instructor.from_gemini(
    client=genai.GenerativeModel(
        model_name="models/gemini-1.5-flash-latest",
    ),
    mode=instructor.Mode.GEMINI_JSON,  # (1)!
)


sound = AudioSegment.from_mp3("sample.mp3")  # (2)!
sound = sound[:60000]


class Transcription(BaseModel):
    summary: str
    exact_transcription: str


resp = client.create(
    response_model=Transcription,
    messages=[
        {
            "role": "user",
            "content": "Please transcribe this recording",
        },
        {
            "role": "user",
            "content": {
                "mime_type": "audio/mp3",
                "data": sound.export().read(),  # (3)!
            },
        },
    ],
)

print(resp)

#> summary='President delivers a speech to a joint session of Congress,
#> highlighting his history in the House of Representatives and thanking
#> the members of Congress for their guidance.',
# >
#> exact_transcription="The President's State of the Union address to a
#> joint session of the Congress from the rostrum of the House of
#> Representatives, Washington DC, January 30th 1961. Mr. Speaker, Mr.
#> Vice-President, members of the Congress, it is a pleasure to return
#> from whence I came. You are among my oldest friends in Washington,
#> and this house is my oldest home. It was here that I first took the
#> oath of federal office. It was here for 14 years that I gained both
#> knowledge and inspiration from members of both"
```

1. Make sure to set the mode to `GEMINI_JSON`, this is important because Tool Calling doesn't work with multi-modal inputs.
2. Use `AudioSegment.from_mp3` to load your audio file.
3. Pass in the audio data as bytes to the `data` field using the content as a dictionary with the right content `mime_type` and `data` as bytes

## Lists of Content

We also support passing in these as a single list as per the documentation for `google-generativeai`. Here's how to do so with a audio segment snippet from the same recording.

Note that the list can contain normal user messages as well as file objects. It's incredibly flexible.

```python
import instructor
import google.generativeai as genai
from pydantic import BaseModel


client = instructor.from_gemini(
    client=genai.GenerativeModel(
        model_name="models/gemini-1.5-flash-latest",
    ),
    mode=instructor.Mode.GEMINI_JSON,  # (1)!
)

mp3_file = genai.upload_file("./sample.mp3")  # (2)!


class Description(BaseModel):
    description: str


content = [
    "Summarize what's happening in this audio file and who the main speaker is",
    mp3_file,  # (3)!
]

resp = client.create(
    response_model=Description,
    messages=[
        {
            "role": "user",
            "content": content,
        }
    ],
)

print(resp)
#> description='President John F. Kennedy delivers State of the Union Address to \
#> Congress. He outlines national challenges: economic struggles, debt concerns, \
#> communism threat, Cold War. Proposes solutions: increased military spending, \
#> new economic programs, expanded foreign aid. Calls for active U.S. role in \
#> international affairs. Emphasizes facing challenges, avoiding panic, and \
#> working together for a better future.'
```

1. Make sure to set the mode to `GEMINI_JSON`, this is important because Tool Calling doesn't work with multi-modal inputs.
2. Upload the file using `genai.upload_file` or get the file using `genai.get_file`
3. Pass in the content as a list containing the normal user message and the file object.