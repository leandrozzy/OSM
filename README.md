<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Gerador de Tática OSM26 - Apenas Prints</title>
  <script src="https://cdn.jsdelivr.net/npm/tesseract.js@4/dist/tesseract.min.js"></script>
  <style>
    body { font-family: Arial; padding: 20px; line-height: 1.6; }
    input, textarea, button { width: 100%; padding: 8px; margin: 6px 0; }
    .container { max-width: 700px; margin:auto; }
    .result, .history { white-space: pre-wrap; background:#f4f4f4; padding:10px; border-radius:5px; margin-top:10px; }
  </style>
</head>
<body>
<div class="container">
  <h1>Gerador de Tática OSM26 - Apenas Prints</h1>

  <!-- Prints do adversário -->
  <h3>Upload dos 3 prints do adversário</h3>
  <input type="file" id="print1" accept="image/*">
  <input type="file" id="print2" accept="image/*">
  <input type="file" id="print3" accept="image/*">

  <!-- Botão -->
  <button onclick="gerarTatica()">Gerar Tática</button>

  <!-- Resultado -->
  <h3>Tática Gerada</h3>
  <div id="output" class="result"></div>

  <!-- Histórico -->
  <h3>Histórico de Jogos</h3>
  <div id="history" class="history"></div>
</div>

<script>
async function lerPrint(file) {
  if(!file) return "";
  const {data: {text}} = await Tesseract.recognize(file, 'eng', {logger: m => console.log(m)});
  return text;
}

// Histórico
function salvarHistorico(entry){
  let hist = JSON.parse(localStorage.getItem("osm26_history") || "[]");
  hist.unshift(entry);
  if(hist.length>20) hist.pop();
  localStorage.setItem("osm26_history", JSON.stringify(hist));
  mostrarHistorico();
}

function mostrarHistorico(){
  let hist = JSON.parse(localStorage.getItem("osm26_history") || "[]");
  let div = document.getElementById("history");
  div.innerHTML = hist.map(h=>`Jogo contra ${h.nome_adv} → ${h.tatica}`).join("\n\n");
}
mostrarHistorico();

// Função principal
async function gerarTatica(){
  const output = document.getElementById("output");
  output.innerText = "Processando OCR... Aguarde...";

  // Ler os 3 prints
  let texto1 = await lerPrint(document.getElementById("print1").files[0]);
  let texto2 = await lerPrint(document.getElementById("print2").files[0]);
  let texto3 = await lerPrint(document.getElementById("print3").files[0]);

  // Combinar os textos
  let textoCompleto = texto1 + "\n" + texto2 + "\n" + texto3;

  // Estimar overalls e formação (pode ajustar manualmente se OCR não pegar corretamente)
  let overall = textoCompleto.match(/Overall:\s*(\d+)/g)?.map(x=>parseInt(x.split(":")[1])) || [90,90,90,90,90];
  let formacao = textoCompleto.match(/\b\d-\d-\d\b/)?.[0] || "4-3-3B";

  // Chance aproximada
  let meuTotal = 450; // você pode alterar conforme seu time
  let advTotal = overall.reduce((a,b)=>a+b,0);
  let chance = Math.round((meuTotal/(meuTotal+advTotal))*100);

  // Prompt para ChatGPT
  let prompt = `
Você é especialista em OSM26. Baseando-se nesta informação extraída dos prints do adversário:

Formação: ${formacao}
Overalls por setor: ${overall.join(", ")}

Gere a tática completa pronta para aplicar no OSM26, incluindo:
- Formação
- Mentalidade
- Pressão
- Ritmo
- Marcação
- Sliders por setor
- Observações
Chance aproximada de vitória: ${chance}%
`;

  // Chamada API
  try{
    const res = await fetch("https://api.openai.com/v1/chat/completions",{
      method:"POST",
      headers:{
        "Content-Type":"application/json",
        "Authorization":"Bearer SEU_TOKEN_OPENAI"
      },
      body: JSON.stringify({
        model:"gpt-4",
        messages:[
          {role:"system", content:"Você é especialista em OSM26."},
          {role:"user", content:prompt}
        ],
        max_tokens:600
      })
    });
    const data = await res.json();
    const text = data.choices && data.choices[0] ? data.choices[0].message.content : "Erro ao gerar tática";
    output.innerText = text;

    salvarHistorico({nome_adv:"Adversário via prints", tatica:text});
  }catch(err){
    output.innerText = "Erro na API: "+err.message;
  }
}
</script>
</body>
</html>
