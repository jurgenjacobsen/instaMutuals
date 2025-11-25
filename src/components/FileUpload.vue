<template>
  <input
    type="file"
    accept=".json"
    @change="handleFileUpload"
    class="rounded-md"
  />
</template>

<script lang="ts">
import { defineComponent, ref, inject } from "vue";
import type { Ref } from "vue";

interface FollowingItem {
  username: string;
  date: Date;
}

interface FollowersItem {
  username: string;
  date: Date;
}

interface PendingItem {
  username: string;
  date: Date;
}

export default defineComponent({
  name: "FileUpload",
  props: {
    fileRequired: String,
  },
  setup(props) {
    const fileContent = ref<string | null>(null);

    // Inject global stores
    const followingGlobal = inject("following") as Ref<any>;
    const followersGlobal = inject("followers") as Ref<any>;
    const notMutualsGlobal = inject("notMutuals") as Ref<any>;
    const sentFollowRequestsGlobal = inject("sentFollowRequests") as Ref<any>;
    const fileErr = inject("fileErr") as Ref<string[]>;

    const removeFileErr = (fileName: string) => {
      const idx = fileErr?.value?.indexOf(fileName);
      if (idx !== -1) fileErr.value.splice(idx, 1);
    };

    const handleFileUpload = (event: Event) => {
      const input = event.target as HTMLInputElement;
      if (!input.files || !input.files[0]) return;

      const file = input.files[0];
      const reader = new FileReader();

      reader.onload = (e) => {
        const result = e.target?.result;

        if (typeof result !== "string") return;

        try {
          const parsedJson = JSON.parse(result);
          fileContent.value = JSON.stringify(parsedJson, null, 2);

          const fileName = input.files?.[0].name;

          // --------------------------
          // FOLLOWING.JSON
          // --------------------------
          if (fileName === "following.json" && props.fileRequired === "following.json") {
            removeFileErr(props.fileRequired);

            followingGlobal.value = {
              fileName,
              data: parsedJson.relationships_following.map((f: any): FollowingItem => {
                const d = f.string_list_data[0];
                return {
                  username: f.title,
                  date: new Date(d.timestamp),
                };
              }),
            };
          }

          // --------------------------
          // FOLLOWERS_1.JSON
          // --------------------------
          else if (fileName === "followers_1.json" && props.fileRequired === "followers_1.json") {
            removeFileErr(props.fileRequired);

            followersGlobal.value = {
              fileName,
              data: parsedJson.map((f: any): FollowersItem => {
                const d = f.string_list_data[0];
                return {
                  username: d.value,
                  date: new Date(d.timestamp),
                };
              }),
            };
          }

          // --------------------------
          // PENDING_FOLLOW_REQUESTS.JSON
          // --------------------------
          else if (
            fileName === "pending_follow_requests.json" &&
            props.fileRequired === "pending_follow_requests.json"
          ) {
            removeFileErr(props.fileRequired);

            sentFollowRequestsGlobal.value = {
              fileName,
              data: parsedJson.relationships_follow_requests_sent.map(
                (f: any): PendingItem => {
                  const d = f.string_list_data[0];
                  return {
                    username: f.title || d.value, // fallback because title is often ""
                    date: new Date(d.timestamp),
                  };
                }
              ),
            };
          }

          // --------------------------
          // INVALID JSON TYPE
          // --------------------------
          else {
            if (Array.isArray(fileErr.value)) {
              fileErr.value.push(props.fileRequired!);
            } else {
              fileErr.value = [props.fileRequired!];
            }

            alert(
              "Invalid JSON file - Please upload following.json, followers_1.json or pending_follow_requests.json"
            );
            fileContent.value = null;
            return;
          }

          // --------------------------
          // COMPUTE NOT MUTUALS
          // --------------------------
          if (followingGlobal?.value?.data && followersGlobal?.value?.data) {
            const following = followingGlobal.value.data.map((f: any) => f.username);
            const followers = followersGlobal.value.data.map((f: any) => f.username);

            const notMutuals = following.filter((u: string) => !followers.includes(u));

            notMutualsGlobal.value = {
              data: notMutuals,
            };
          }
        } catch (err) {
          alert("Invalid JSON file");
          fileContent.value = null;
        }
      };

      reader.onerror = () => {
        alert("Error reading file");
        fileContent.value = null;
      };

      reader.readAsText(file);
    };

    return {
      fileContent,
      handleFileUpload,
    };
  },
});
</script>
