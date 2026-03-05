We have an issue.

As we keep adding more and more things onto this pipeline for processing text, improvements and detections it's getting more and more complicated to get things right. The tables and joins are getting more and more complex

My proposed solution. In a nut shell...
Despite all the old messy tables and processing... the end result will be clean and readable.

processed_paragraphs that links back to chapter ID and not to a paragraph reference.
after the final stage of our pipe line regarding text manipulation and highlighting we get llm_paragraphs - I want to change this table to be dialogue_paragraphs (or something similiar)...

a new table finalised_paragraphs, which will transfer and accumulate all the data from each stage of the process... be it, character relations or table / header identification.

each row will join to a finalised_paragraph_tts table, which will contain a the timing data for the tts, and we'll generate them per paragraph. If however we generate the TTS doesn't have timing data, we'll leave that as well. but still mark it as complete.

Once the processing of all chapters is complete, we'll stitch them all together with say a 200ms (we'll use a constant) gap between each paragraph and save that as a new file, and we can save that somewhere.... Your suggestions welcome.

The front end will pull from this finalised_paragraph_tts table for display and it'll join onto the tts timing table. And pull all the records at the start, this will remove completely any confusion between characters ect...

This will give us power to use different sources for the character audio ect, so long as we know the duration of the clip of audio we stitch together, we'll be able to quickly rebuild the tts from any issues without having to re do the whole thing.


