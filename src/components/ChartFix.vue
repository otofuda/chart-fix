<script lang="ts" setup>
import { ref } from 'vue';

// interface ChartData {
//   raku: NoteData[];
//   easy: NoteData[];
//   normal: NoteData[];
//   hard: NoteData[];
//   extra: NoteData[];
// }

type ChartData = Record<string, NoteData[]>;

interface NoteData {
  type: number;
  measure: number;
  lane: 1 | 2 | 3 | 4 | 5 | -1;
  position: number;
  split: number;
  option: NoteOptionData[];
  end: NoteData[];
}

type NoteOptionData = number | string;

// Type絞り込みの設定
const types = [1, 2, 3, 4, 5, 94, 95, 96, 97, 98, 99, 100];
const selectedTypes = ref<number[]>([...types]);

// 型チェックの設定
const checkOptionString = ref(true);

const messages = ref<string[]>([]);
let chartObject: ChartData | null = null;

let file: File | null = null;

const onChange = (event: Event) => {
  const target = event.currentTarget as HTMLInputElement;
  if (target.files && target.files[0]) {
    file = target.files[0];
    messages.value.push("ファイル読込完了: " + file.name);

    try {
      file.text().then((text) => {
        chartObject = JSON.parse(text);
        console.log(chartObject);
        messages.value.push("譜面パース完了");
      }).catch((error) => {
        messages.value.push("譜面パースエラー: " + error);
      });
    } catch (error) {
      messages.value.push("データ読み込みエラー: " + error);
    }
  }
}

const onConvert = () => {
  if (!file || !chartObject) {
    messages.value.push("譜面の読み込みが完了していません");
    return;
  }

  const newObject: ChartData = {
    raku: [],
    easy: [],
    normal: [],
    hard: [],
    extra: []
  };
  ["raku", "easy", "normal", "hard", "extra"].forEach((difficulty) => {
    const noteArray = chartObject && chartObject[difficulty];
    if (noteArray === null) return;


    // 特定ノーツの削除
    const before = noteArray.length;
    let after = noteArray.length;
    newObject[difficulty] = noteArray.filter((note) => {
      // selectedTypeに合致するか
      const isSelectedType = selectedTypes.value.includes(note.type);

      const result: boolean = isSelectedType;
      if (result === false) after--;
      return result;
    });

    messages.value.push(`Filterを実行: [${difficulty}] ${before} => ${after} Objects`);

    // option を string[] に強制
    const fixOptionToString = (note: NoteData): NoteData => {
      const newNote: NoteData = note;
      newNote.option = note.option.map((opt, j) => {
        if (typeof opt !== "string") {
          messages.value.push(
            `不正な型のOptionを発見: [${difficulty}] m: ${note.measure} (${note.position}/${note.split}), option[${j}]`
          );
          return String(opt);
        }
        return opt;
      });
      // end配列を再帰的にチェック
      newNote.end = newNote.end.map((end) => fixOptionToString(end));
      return newNote;
    }

    // 型チェックを実行
    newObject[difficulty] = newObject[difficulty].map((note) => {
      let newNote = note;

      // note.option : string[]
      if (checkOptionString.value) {
        newNote = fixOptionToString(newNote);
      }

      return newNote;
    });
  });


  console.log(newObject);

  const blob = new Blob([JSON.stringify(newObject, null, 4)], {
    type: "application/json"
  });
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = file.name;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  messages.value.push(`ファイルを保存: ${file.name}`);
}
</script>

<template>
  <v-container>
    <p class="mb-4">譜面データ修復ツール V2 (仮)</p>

    <v-row>
      <v-col>
        <v-file-input label="譜面ファイルを選択" @change="onChange" />

        <p class="mb-4">Filter by note type</p>


        <v-row class="mb-4">
          <v-checkbox
            v-model="selectedTypes"
            dense
            hide-details
            :label="`Type: ${type}`"
            :value="type"
            v-for="type in types"
          />
        </v-row>

        <p class="mb-4">Type check</p>

        <v-row class="mb-4">
          <v-checkbox
            v-model="checkOptionString"
            dense
            hide-details
            label="note.option : string[]"
          />
        </v-row>

        <v-btn block variant="outlined" @click="onConvert">
          変換を実行する
        </v-btn>
      </v-col>
      <v-col style="max-width: 500px">
        <v-card tile class="mb-4">
          <v-list-item v-for="m in messages">
            <v-list-item-header>
              <v-list-item-title>{{ m }}</v-list-item-title>
            </v-list-item-header>
          </v-list-item>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>
