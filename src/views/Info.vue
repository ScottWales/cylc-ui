<!--
Copyright (C) NIWA & British Crown (Met Office) & Contributors.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<!--
  This is a bare-bones text-based tree view that displays live workflow data.

  It is intended to serve as a basis for future live-view development and to
  serve as functional documentation for view writing.

  NOTE:
    This view is not available in "production" mode.
-->

<template>
  <div class="c-info">
    <!-- Your HTML here -->
    <!-- Get data from the "task" object -->
    <ul v-if="requestedTokens && task.id">
      <li>
        <span>Cycle:</span>
        <span>{{ requestedTokens.cycle }}</span>
      </li>
      <li>
        <span>Task:</span>
        <span>{{ requestedTokens.task }}</span>
      </li>
      <li>
        <span>Title:</span>
        <span>{{ task.task?.meta?.title }}</span>
      </li>
      <li>
        <span>Description:</span>
        <span>{{ task.task?.meta?.description }}</span>
      </li>
      <li>
        <span>URL:</span>
        <span>{{ task.task?.meta?.URL }}</span>
      </li>
    </ul>
  </div>
</template>

<script>
import gql from 'graphql-tag'
import { mapGetters } from 'vuex'
import { getPageTitle } from '@/utils/index'
import graphqlMixin from '@/mixins/graphql'
import subscriptionComponentMixin from '@/mixins/subscriptionComponent'
import SubscriptionQuery from '@/model/SubscriptionQuery.model'
import DeltasCallback from '@/services/callbacks'
import {
  initialOptions,
  useInitialOptions
} from '@/utils/initialOptions'

// Any fields that our view will use (e.g. TaskProxy.status) must be requested
// in the query.
// Most of this is just boilerplate the important thing is the three declarations
// at the end.
const QUERY = gql`
subscription SimpleTreeSubscription ($workflowId: ID, $taskID: ID) {
  deltas(workflows: [$workflowId]) {
    added {
      ...AddedDelta
    }
    updated (stripNull: true) {
      ...UpdatedDelta
    }
    pruned {
      ...PrunedDelta
    }
  }
}

fragment AddedDelta on Added {
  workflow {
    ...WorkflowData
  }
  taskProxies(ids: [$taskID]) {
    ...TaskProxyData
  }
  jobs {
    ...JobData
  }
}

fragment UpdatedDelta on Updated {
  workflow {
    ...WorkflowData
  }
  taskProxies(ids: [$taskID]) {
    ...TaskProxyData
  }
  jobs {
    ...JobData
  }
}

# We must list all of the types we request data for here to enable automatic
# housekeeping.
fragment PrunedDelta on Pruned {
  workflow
  taskProxies
  jobs
}

# We must always request the reloaded field whenever we are requesting things
# within the workflow like tasks, cycles, etc as this is used to rebuild the
# store when a workflow is reloaded or restarted.
fragment WorkflowData on Workflow {
  id
  reloaded
}

# We must always request the "id" for ALL types.
# The only field this view requires beyond that is the status.
fragment TaskProxyData on TaskProxy {
  # "TaskProxy" fields (these provide info about a task instance)
  id
  task {
    # "Task" field (these provide info about the task config)
    meta {
      title
      description
      URL
    }
  }
}

# Same for jobs.
fragment JobData on Job {
  id
}
`

/** Callback for assembling the log file from the subscription */
class InfoCallback extends DeltasCallback {
  /**
   * @param {Results} results
   */
  constructor (task) {
    super()
    this.task = task
  }

  onAdded (added, store, errors) {
    Object.assign(this.task, added.taskProxies[0])
  }

  onUpdated (updated, store, errors) {
    if (updated?.taskProxies) {
      Object.assign(this.task, updated.taskProxies[0])
    }
  }

  onPruned (pruned) {
    // TODO: delete task proxy data when it drifts out of the window
    // Note: we can keep the taskProxy.task data, that's still valid
  }
}

export default {
  name: 'InfoView',

  // These mixins enable various functionalities.
  mixins: [
    graphqlMixin,
    subscriptionComponentMixin
  ],

  // This sets the page title.
  head () {
    return {
      title: getPageTitle('App.workflow', { name: this.workflowName })
    }
  },

  props: {
    initialOptions,
  },

  setup (props, { emit }) {
    const requestedTokens = useInitialOptions('requestedTokens', { props, emit })
    return {
      requestedTokens
    }
  },

  data () {
    return {
      // This is the task we will request metadata for.
      // when you change these values, the old query will be automatically canceled
      // and re-issued with the new values
      requestedCycle: '1',
      requestedTask: 'foo',
      task: {},
    }
  },

  computed: {
    ...mapGetters('workflows', ['getNodes']),

    // List of workflow IDs we are displaying.
    // NOTE: Write all views to be multi-workflow capable.
    // NOTE: workflowId is provided by a mixin.
    workflowIDs () {
      return [this.workflowId]
    },

    // This registers the query with the WorkflowService, once registered, the
    // WorkflowService promises to make the data defined by the query available
    // in the store and to keep it up to date.
    query () {
      return new SubscriptionQuery(
        QUERY,
        { ...this.variables, taskID: this.requestedTokens?.relativeID },
        `info-query-${this._uid}`,
        [
          new InfoCallback(this.task)
        ],
        /* isDelta */ true,
        /* isGlobalCallback */ false
      )
    }
  }

}
</script>

<style scoped lang="scss">
  .c-info {
    // your SCSS here (SCSS is a superset of CSS)
  }
</style>
