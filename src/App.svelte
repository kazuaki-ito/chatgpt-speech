<script>
  import {Configuration, OpenAIApi} from 'openai'

  const OPENAI_API_KEY = 'your key'

  const configuration = new Configuration({
    apiKey: OPENAI_API_KEY,
  })
  const openai = new OpenAIApi(configuration)

  const speechSynthesis = window.speechSynthesis
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  if (!speechSynthesis || !SpeechRecognition) {
    throw new Error('このブラウザは対応していません')
  }
  const recognition = new SpeechRecognition()
  recognition.lang = "ja-JP"
  recognition.continuous = true

  let tmp = ''
  recognition.onresult = async function (event) {
    console.info('onresult')
    if (speechSynthesis.speaking || isAsking) return
    if (event.results.length > 0) {
      const result = event.results[event.results.length - 1]
      tmp += result[0].transcript
      if (result.isFinal) {
        if (!tmp) return
        val += '<br/>' + `you:${tmp}`
        await askGpt(tmp)
        tmp = ''
      }
    }
  }

  let val = ''
  let isAsking = false
  let stareted = false
  const askGpt = async (content) => {
    speechSynthesis.cancel()
    const prev = val
    try {
      isAsking = true
      val += '<br/>' + '処理中...'
      const response = await openai.createChatCompletion({
        model: 'gpt-3.5-turbo',
        messages: [
          {role: "user", content}
        ],
      })
      console.info(response.data.choices[0].message)
      const answer = response.data.choices[0].message.content
      val = prev + '<br/>' + `gpt:${answer}` + '<br/>'
      speak(answer)
      isAsking = false
    } catch (e) {
      console.error(e)
      val = prev + '<br/>' + `gpt:error` + '<br/>'
      isAsking = false
    }
  }
  const speak = (content) => {
    const uttr = new SpeechSynthesisUtterance()
    uttr.lang = 'ja-JP'
    uttr.volume = 1.0
    uttr.text = content
    uttr.rate = 1.0
    uttr.pitch = 1.0
    speechSynthesis.speak(uttr)
    console.info('speak! - ' + content)
  }
  const startSpeechRecognition = () => {
    console.info('startSpeechRecognition')
    recognition.start()
    stareted = true
    console.info('recognition.start')
  }
  const handleStart = () => {
    startSpeechRecognition()
  }
</script>

<main>
    <h1>history</h1>
    質問をマイクにむけて話すとChatGptが回答してくれるだけのページ<br/>
    {#if !stareted}
        <a href="#" on:click={handleStart}>start</a><br/>
    {/if}
    {@html val}
</main>

<style>
    main {
        text-align: left;
        padding: 1em;
    }
</style>