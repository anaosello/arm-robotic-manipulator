# A.R.M. — Parâmetros extraídos para URDF (v0.1)

**Fonte:** `ARM_RPP_v01.FCStd` (FreeCAD). **Status:** NÃO VALIDADO PARA VOO/FABRICAÇÃO.
**Unidades finais:** metros, radianos, kg, kg·m². Conversão de mesh mm→m (×0,001).
**Pose de referência (home, q=0):** θ1=0, d2=0, d3=0,100 m.

Backup do CAD original preservado em `ARM_RPP_v01_BACKUP_20261112.FCStd`.
O projeto mecânico **não foi alterado** para esta exportação.

---

## 1. Estrutura cinemática (RPP, 3 GDL atuados)

Cadeia: `base_link → [J1 rev] → link1 → [J2 pris] → link2 → [J3 pris] → link3 → [fixa] → tcp`

| Junta | Tipo | Eixo | Origem (no pai) | Variável | Limites (CAD) |
|-------|------|------|-----------------|----------|---------------|
| joint1 | revoluta | z (0 0 1) | (0,0,0) | q1 = θ1 | −1,570796 … +1,570796 rad (−90°…+90°) |
| joint2 | prismática | z (0 0 1) | (0,0,0) | q2 = d2 | 0 … 0,100 m |
| joint3 | prismática | y (0 1 0) | (0,0,0) | q3 = d3 − 0,100 | 0 … 0,200 m (d3 = 0,100…0,300 m) |
| tcp_joint | fixa | — | (0; 0,100; 0,120) em link3 | — | efetuador (face do eletroímã) |

**Todas as origens das juntas são identidade:** no CAD os offsets cinemáticos são
nulos em home e a geometria carrega o posicionamento espacial. Isso reproduz a FK
exatamente (ver §4).

**Cinemática direta preservada:**
`x = −d3·sen(θ1)`, `y = d3·cos(θ1)`, `z = l1 + d2`, com **l1 = 0,120 m** e `d3 = 0,100 + q3`.

O eletroímã é o **efetuador** (não é junta atuada): sua geometria faz parte de `link3`;
o frame `tcp` marca a face de contato.

---

## 2. Composição dos elos (quais peças em cada link)

| Link | Peças (objetos FreeCAD) |
|------|-------------------------|
| **base_link** | PlacaBase, Alojamento, Rolamento1*, Rolamento2*, MotorXM430† |
| **link1** | Disco, Eixo, ColunaVert, GuiaVert1*, GuiaVert2*, AtuadorVert† (T16-100) |
| **link2** | CarroPlaca, SuporteHoriz, GuiaHoriz*, CorpoT16_200†, VigaMovel, AtuadorMont1/2 (brackets) |
| **link3** | SuporteIma, Eletroima†, InterfFerramenta, CarroHoriz, Riser, AtuadorGarra (garra) |

`*` = comercial **sem massa de catálogo no projeto** (guias igus, rolamentos SKF) →
incluídos na **mesh (visual/colisão)**, porém **excluídos da inércia** (massa desconhecida).
`†` = comercial com **massa de catálogo**; inércia aproximada pelo **envelope** (não é a
distribuição real de massa).

---

## 3. Parâmetros dinâmicos por elo

Densidades do projeto: **Al 6061-T6 = 2700 kg/m³ (0,0027 g/mm³)**, **aço = 7850 kg/m³ (0,00785 g/mm³)**.
Inércia expressa no frame do elo (coincide com o global em home), com `<origin>` no CoM.

| Link | Massa (kg) | CoM (m) [x,y,z] | ixx | iyy | izz | ixy | ixz | iyz (kg·m²) |
|------|-----------|-----------------|-----|-----|-----|-----|-----|-----|
| base_link | 0,9875 | [0; 0; 0,02663] | 1,070e-3 | 1,068e-3 | 1,050e-3 | 0 | 0 | 0 |
| link1 | 0,8490 | [0; −0,00397; 0,16572] | 7,643e-3 | 8,376e-3 | 9,528e-4 | 0 | 0 | 1,234e-4 |
| link2 | 0,6040 | [0,04618; 0,13638; 0,16168] | 7,089e-3 | 8,319e-4 | 7,632e-3 | −7,286e-4 | −4,466e-5 | −2,036e-4 |
| link3 | 0,2187 | [0,01564; 0,10259; 0,15422] | 1,624e-4 | 3,333e-4 | 2,525e-4 | 8,873e-6 | −7,848e-5 | 6,565e-6 |

**Massa total modelada = 2,659 kg.**

Valores recalculados após o refino geométrico (v0.1 realista): as peças custom passaram a
ter geometria com rebaixos/furos de fixação e envelopes mais fiéis, e o **atuador vertical
Actuonix T16-100 (138 g, catálogo)** foi incluído em `link1` como `AtuadorVert` (resolve a
pendência §5.2). O **servo de rotação XM430 foi realojado dentro do pedestal** (não mais
pendurado sob a base) e o **atuador horizontal T16-200 foi ancorado à estrutura** por dois
brackets (`AtuadorMont1/2`, link2) e acionado por uma garra no carro móvel (`AtuadorGarra`,
link3), eliminando o aspecto "solto". A massa total subiu de 1,689 kg para 2,659 kg, mais próxima da estimativa
de inventário do memorial (~2,14 kg estrutura; comerciais sem catálogo ainda ausentes).

---

## 4. Validação executada

- **Validador autônomo** `validate_urdf.py` (numpy): XML OK; raiz única `base_link`;
  árvore sem ciclos; tipos/eixos das 3 juntas conferem; **FK do URDF × fórmula RPP:
  erro máx = 2,8e-17 m** (precisão de máquina) sobre varredura θ1∈{−90,−45,0,45,90}°,
  d2∈{0; 0,1}, d3∈{0,1; 0,2; 0,3}; todas as meshes referenciadas existem.
- **Independência dos 3 movimentos:** confirmada — cada junta afeta apenas seu GDL
  (a igualdade exata com `(−d3·senθ, d3·cosθ, l1+d2)` prova desacoplamento).
- **Robotics Toolbox:** `test_rtb_swift.py` **executado com sucesso** (RTB em Python 3.14):
  `rtb.Robot.URDF()` importa o modelo, reporta `estrutura = 'RPP'`, `n = 3` juntas atuadas
  (Rz·tz·ty) e **FK RTB × fórmula RPP: erro máx = 0,000 m**. Visualização Swift opcional
  via `python test_rtb_swift.py --swift`.

---

## 5. Dados FALTANTES / pendências (não inventados)

1. **Massa/inércia de comerciais sem catálogo no projeto:** guias igus (NS-01-27 ×2,
   NS-01-40), patins, rolamentos SKF 6001 ×2. Hoje entram só como geometria (visual),
   massa = 0 → **massa dos elos é subestimada**.
2. **Atuador vertical Actuonix T16-100 (138 g):** ~~não havia sólido no CAD~~ **RESOLVIDO
   na v0.1 realista:** modelado como `AtuadorVert` em `link1` (massa de catálogo 138 g,
   inércia por envelope). Importar STEP oficial para inércia real.
3. **Inércia de comerciais com catálogo** (XM430, T16-200, eletroímã): aproximada pelo
   **envelope** (massa certa, distribuição não). Importar STEP oficial para inércia real.
4. **Esforço/velocidade das juntas no `<limit>`:** vêm de catálogo (T16: 90 N, 0,018 m/s;
   XM430: 0,82 N·m contínuo). **Velocidade de joint1 (4,8 rad/s) é nominal — CONFIRMAR.**
   Os limites de **posição**, esses sim, vêm do CAD.
5. **Fixadores e fiação:** sem massa/geometria.
6. **Interface/furação OEM ao DJI Matrice 400** e transformação fixa do frame local para
   o referencial do drone: indefinidas (a base do URDF está no eixo de rotação, z p/ cima).

---

## 6. Arquivos

```
models/
  arm.urdf                 # URDF (4 links + tcp, 3 juntas atuadas)
  meshes/
    base_link.stl  link1.stl  link2.stl  link3.stl   # em metros
  validate_urdf.py         # validação autônoma (roda no ambiente atual)
  test_rtb_swift.py        # teste na Robotics Toolbox + Swift (ambiente Py<=3.12)
  PARAMETROS_URDF.md       # este documento
```
