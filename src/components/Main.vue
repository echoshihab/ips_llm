<template>
  <div>
    <v-textarea label="Report Content" min-width="800px"></v-textarea>
    <v-btn @click="submitPrompt"> Submit </v-btn>
  </div>
  <div class="progression-bar-container" v-if="progressionBarOn">
    <div>{{ progressBarContent }}</div>
    <v-progress-linear indeterminate></v-progress-linear>
  </div>


</template>

<script setup>
import OpenAI from "openai";
import { GoogleGenerativeAI } from "@google/generative-ai";
import { ref, computed } from 'vue';

// configuration
const openai = new OpenAI({ apiKey: import.meta.env.VITE_OPEN_AI_KEY, dangerouslyAllowBrowser: true });
const genAI = new GoogleGenerativeAI(import.meta.env.VITE_GEMINI_KEY);

let exampleContent1 = "Sarah Lee, a 28-year-old female, presents with a history of bipolar disorder. She's currently on Lithium 600mg twice daily and \
        Quetiapine 200mg nightly for mood stabilization. During our consultation, she disclosed a past allergic reaction to Lamotrigine, \
        characterized by a severe rash and fever. Given this, Lamotrigine should be avoided in her treatment regimen to prevent recurrence of adverse reactions."

let medications = ref([]);

const stringifiedMedications = computed(() => {
  if (medications.value.length) {
    return medications.value.map(m => JSON.stringify(m))
  }
})

let allergies = ref([]);
const stringifiedAllergies = computed(() => {
  if (allergies.value.length) {
    return allergies.value.map(a => JSON.stringify(a))
  }
})

let progressionBarOn = ref(false);
let progressBarContent = ref("");

async function submitPrompt() {

  await retrieveResourcesFromOpenAI();
  await validateResourcesAgainstGemini();

}

async function retrieveResourcesFromOpenAI() {

  progressionBarOn.value = true;
  progressBarContent.value = "Retrieving IPS FHIR Resources from OpenAI..."

  const completion = await openai.chat.completions.create({
    messages: [
      {
        "role": "system",
        "content": `You are an expert FHIR resource creating algorithm who parses unstructured patient report content that physicians write,\
        and creates valid medication and allergy intolerance resources according to FHIR International Patient Summary (IPS) profile. Ensure \
        that json response contains a key of medications which includes an array of the medication resources and a key of allergies which \
        contains an array of allergy intolerance resources. The arrays should contain all medication and allergies retrieved from the content \
        Here is the report: ${exampleContent1}`
      },
    ],
    model: "gpt-3.5-turbo",
    response_format: { "type": "json_object" }
  });

  medications.value = JSON.parse(completion.choices[0].message.content).medications
  allergies.value = JSON.parse(completion.choices[0].message.content).allergies

  console.log(medications.value)
  console.log(allergies.value)

  progressBarContent.value = "";
  progressionBarOn.value = false;
}

async function validateResourcesAgainstGemini() {

  progressionBarOn.value = true;
  progressBarContent.value = "Validating FHIR Resources via Gemini..."

  // For text-only input, use the gemini-pro model
  const model = genAI.getGenerativeModel({ model: "gemini-pro" });

  const prompt = `You are an expert FHIR International Patient Summary profile (IPS) validation algorithm. An unstructured patient report content \
  will be provided, and the IPS profile compliant FHIR medication and allergy intolerance resources that were generated via another tool from this \
  content. You are to verify the following: 1. if the generated medication and allergy intolerance resources are valid according to IPS profile \
  2. The correct number of medication and allergy resources are present and 3. Ensure that allergy is not listed as medication resource and medication \
 is not listed as allergyIntolerance resource.\
 Here is the content:\
${exampleContent1}

 Medications:
 ${stringifiedMedications.value.join(', ')}

 Allergies:
 ${stringifiedAllergies.value.join(', ')}

Respond in json with a key of 'valid': where the value can either be true or false and a key of 'MedicationIssues' and 'AllergyIssues' where the issues with the resources are described if any. 
Return empty value for key of 'issues' if the resources are valid.`

  const result = await model.generateContent(prompt);
  const response = await result.response;
  const text = response.text();
  console.log(text);

  progressBarContent.value = "";
  progressionBarOn.value = false;
}

</script>



<style scoped>
.progression-bar-container {
  margin-top: 30px;
}
</style>
