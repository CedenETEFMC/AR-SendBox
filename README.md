# Caixa de Areia de Realidade Aumentada – Jogo Interativo  

Este repositório contém o código completo de um **jogo de realidade aumentada** desenvolvido com o sensor **Kinect v1** e um projetor multimídia. O jogo transforma uma caixa de areia em uma experiência interativa, onde as crianças moldam a areia para alcançar objetivos definidos pelo sistema.  

⚠️ **Importante:**  
- Este projeto **funciona apenas com o Kinect v1**.  
- Não é compatível com o Kinect v2.  
- O código foi desenvolvido para rodar exclusivamente em **macOS**.  

## Funcionamento  
- O **Kinect v1** captura a profundidade da superfície da areia.  
- O código converte as distâncias em **cores*seguindo o gradiente:  
  - Violeta → nível mais baixo  
  - Anil  
  - Azul  
  - Verde  
  - Amarelo  
  - Laranja  
  - Vermelho → nível mais alto  
- O projetor exibe essas cores diretamente sobre a areia, criando um mapa topográfico dinâmico.  

## Regras do Jogo  
1. A projeção é dividida em 4 quadrantes.  
2. O sistema sorteia uma cor para cada quadrante e projeta sobre a areia.  
3. As crianças têm **40 segundos** para moldar a areia de forma que cada quadrante fique predominantemente da cor sorteada.  
4. Após o tempo:  
   - O código lê a cor predominante em cada quadrante.  
   - O sistema exibe na tela qual cor foi detectada em cada quadrante.  
   - Em seguida, verifica se houve acerto ou erro:  
     - Se **todos acertarem**, a projeção pisca em verde indicando vitória.  
     - Se **pelo menos um quadrante estiver errado**, a projeção pisca em vermelho indicando derrota.  

## Como Executar  
- Instale as dependências necessárias.  
- Certifique-se de ter instalado o **libfreenect**, **OpenGL** e **GLUT**, além do compilador **clang**.  
- Conecte o Kinect v1 ao computador.  
- Verifique se o dispositivo foi reconhecido corretamente pelo sistema.  
- Abra o terminal.  
- No macOS, abra o **Finder**, localize a pasta do projeto e arraste-a para o terminal.  
- Isso fará aparecer o caminho completo da pasta no terminal.  
- Acesse o diretório do projeto.  
- Digite `cd` antes do caminho exibido no terminal para entrar na pasta do código.  
- Compile o código:  
  ```bash
  clang -v NOME_DO_ARQUIVO.c -o NOME_DO_ARQUIVO -I./../include/libfreenect -L./../lib -lfreenect -framework OpenGL -framework GLUT
  ```  
- Execute o programa:  
  ```bash
  ./NOME_DO_ARQUIVO
  ```  
  ⚠️ Lembre-se: o nome após `./` deve ser exatamente o mesmo que você usou no parâmetro `-o` na compilação.  
- Inicie a partida:  
  - Após o jogo abrir, pressione a tecla **espaço** para iniciar a partida.  
  - Toda partida deve ser iniciada manualmente pelo espaço.  

## Calibragem do Kinect  
Para garantir uma leitura correta da profundidade e uma projeção alinhada, é necessário calibrar o Kinect ajustando os valores mínimos e máximos de distância.  

No código, essa configuração é feita pelas variáveis:  

```c
const float MIN_DEPTH = 859.0f;
const float MAX_DEPTH = 949.0f;
```  

### O que significam  
- **MIN_DEPTH** → distância mínima que o Kinect começa a considerar na leitura.  
- **MAX_DEPTH** → distância máxima que o Kinect considera antes de parar a leitura.  

### Como calibrar  
- Se o Kinect estiver **muito próximo** da caixa de areia, diminua os valores.  
- Se o Kinect estiver **mais distante**, aumente os valores.  
- Ajuste até que a projeção das cores corresponda corretamente ao relevo da areia.  

### ⚠️ Observação  
Alterar esses valores permite **mover o Kinect** sem comprometer a qualidade da leitura e da projeção. É importante testar diferentes configurações até encontrar a faixa ideal para o seu ambiente.  

