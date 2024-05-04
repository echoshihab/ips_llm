<template>
  <div>

    <v-textarea label="Report Content" min-width="800px"></v-textarea>
    <v-btn @click="submitPrompt"> Submit </v-btn>
  </div>
  <div class="progression-bar-container">
    <div>{{ progressBarContent }}</div>
    <v-progress-linear indeterminate></v-progress-linear>
  </div>


</template>

<script setup>
import OpenAI from "openai";
import { GoogleGenerativeAI } from "@google/generative-ai";
import { ref } from 'vue';

const openai = new OpenAI({ apiKey: import.meta.env.VITE_OPEN_AI_KEY, dangerouslyAllowBrowser: true });

// gemini 
const genAI = new GoogleGenerativeAI(import.meta.env.VITE_GEMINI_API_KEY);


let medications = ref([]);
let allergies = ref([]);
let progressionBarOn = ref(false);
let progressBarContent = ref("");

async function submitPrompt() {

  const completion = await openai.chat.completions.create({
    messages: [
      {
        "role": "system",
        "content": "You are an expert FHIR resource creating algorithm who parses unstructured patient report content that physicians write,\
        and creates valid medication and allergy intolerance resources according to FHIR International Patient Summary (IPS) profile. Ensure \
        that json response contains a key of medications which includes an array of the medication resources and a key of allergies which \
        contains an array of allergy intolerance resources. The arrays should contain all medication and allergies retrived from the content \
        Here is the report: Sarah Lee, a 28-year-old female, presents with a history of bipolar disorder. She's currently on Lithium 600mg twice daily and \
        Quetiapine 200mg nightly for mood stabilization. During our consultation, she disclosed a past allergic reaction to Lamotrigine, \
        characterized by a severe rash and fever. Given this, Lamotrigine should be avoided in her treatment regimen to prevent recurrence of adverse reactions.\
        "
      },
    ],
    model: "gpt-3.5-turbo",
    response_format: { "type": "json_object" }
  });


  medications = JSON.parse(completion.choices[0].message.content).medications
  allergies = JSON.parse(completion.choices[0].message.content).allergies

  console.log(medications)
  console.log(allergies)
}

</script>



<style scoped>
.progression-bar-container {
  margin-top: 30px;
}
</style>
